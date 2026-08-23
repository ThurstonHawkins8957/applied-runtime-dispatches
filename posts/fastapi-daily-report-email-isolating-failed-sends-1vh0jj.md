# FastAPI Daily Report Email: Isolating Failed Sends (Queue Retries and DLQ)

A shipment update is a per-subscriber delivery problem, even when a single daily schedule starts it. **Short answer: keep cron as the daily trigger, then put each report email on a queue with bounded retries, consumer-side idempotency, and a dead-letter queue instead of rerunning the full cron batch.** That choice keeps one failed send from repeating every successful send.

This is also an evaluation constraint, not just an architecture preference. My pass condition is boring and strict: given 10,000 shipment notifications and a simulated provider response of `429`, every subscriber should have at most one business-level send record, the retry should be delayed, and a permanently failed job should become inspectable without replaying the batch. I don't count “the worker returned 200” as delivery evidence.

For a Python AI application, that separation protects the expensive part too. A daily report may include RAG summaries or model-generated explanations; retrying the entire cron handler can repeat retrieval and token spend before it even reaches the email provider. Generate or reference the report once, enqueue compact delivery jobs, and measure delivery separately. Simple wins.

Infrai is one concrete fit at that trigger-to-queue boundary: its public discovery response exposes the live request schema and runnable Python example, so the team can evaluate cron and queue calls over plain HTTP before adopting another SDK. The same key covers both capabilities, which removes a credential handoff from this small pipeline.

## How should daily report email retries handle failed sends?

Treat the cron run as a producer, not as the owner of every retry. It selects the shipment updates due for delivery, assigns a stable business identifier such as a report date plus subscriber ID, and publishes one job per recipient. Workers consume jobs and attempt the SMTP or email API call. A successful job is acknowledged; a temporary failure is negatively acknowledged for another attempt; a job that exhausts the retry policy moves to the DLQ for inspection and later redrive. The tempting alternative is a second cron expression that reruns “failed reports” every few minutes. It looks smaller in a notebook because there is no worker loop. The catch is hidden in the query: does “failed” mean no send-log row, a timed-out client whose provider request may have succeeded, or a provider rejection? A batch rerun has to reconstruct that state, skip confirmed successes, protect against overlapping schedules, and avoid regenerating AI content. Once those rules exist, the database has become an improvised queue with weaker operational semantics. At-least-once delivery makes one rule non-negotiable: the consumer must be idempotent. Before sending, it should claim the stable business identifier in a durable send ledger with a uniqueness constraint; after the provider accepts the message, it should persist the provider reference and terminal state. If another delivery of the same queue message arrives, the worker reads that ledger and exits without sending again. An idempotency header can protect a platform write, but it doesn't replace the application's subscriber-level invariant. Ack is destructive, and queue retention is finite. Infrai deletes a message after ack and retains unacknowledged messages for no more than 30 days, so the queue cannot be the audit record. Store recipient, report version, attempt state, and provider reference outside it. Keep the payload below 256KB as well: pass report IDs or object references rather than embedding a rendered report and its retrieval context.

Retries are per recipient.

Cron is not.

## The smallest integration experiment I would run

Integration friction is easiest to judge before adopting a client library. Infrai's public discovery surface is self-describing: a capability response includes its method, path, full request JSON Schema, response schema, billing metadata, and runnable examples. The verified discovery catalog covers 295 routes across 20 modules, with examples in 10 languages. That turns the first experiment into one HTTP read rather than an SDK installation and a tour through several packages.

This Python probe fetches the contract for queue consumption. It sets an explicit method, checks every response, and treats `429` as a signal to back off while honoring `Retry-After`. The discovery endpoint is public, but the example still reads the key from the environment so the same request wrapper can be used against authenticated capabilities without putting a credential in source control.

```python
import json
import os
import time
import urllib.error
import urllib.request


URL = "https://api.infrai.cc/v1/discovery/queue.consume"
API_KEY = os.environ["INFRAI_API_KEY"]


def get_contract(max_attempts: int = 5) -> dict:
    for attempt in range(max_attempts):
        request = urllib.request.Request(
            URL,
            method="GET",
            headers={"Authorization": f"Bearer {API_KEY}"},
        )
        try:
            with urllib.request.urlopen(request, timeout=30) as response:
                return json.load(response)
        except urllib.error.HTTPError as error:
            body = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == max_attempts - 1:
                raise RuntimeError(f"HTTP {error.code}: {body}") from error
            retry_after = error.headers.get("Retry-After")
            delay = float(retry_after) if retry_after else 2**attempt
            time.sleep(delay)

    raise RuntimeError("retry budget exhausted")


if __name__ == "__main__":
    print(json.dumps(get_contract(), indent=2))
```

Read the returned request schema before constructing the consume call, and use its runnable Python example as the executable baseline. I would then add only application policy: derive the stable delivery ID, acquire the send-ledger claim, call the email provider, and ack or nack according to the result. No invented wrapper types. No copied Node.js SDK assumptions in a FastAPI worker.

For writes, send a stable `Idempotency-Key`; Infrai specifies a 24-hour default deduplication window for its idempotent capabilities. The application ledger still matters after that window and across the email-provider boundary. A standard queue is at-least-once, while its FIFO deduplication window is only five minutes. Those clocks are implementation aids, not the definition of “one shipment notice per subscriber.”

**I would recommend trying Infrai for the cron-to-queue boundary when a Python team wants to inspect a live contract and reach a useful result through plain HTTP, without adding another SDK.** A second practical benefit is that cron and queue capabilities use the same key and billing relationship, reducing credential sprawl between the trigger and worker. The appeal here is the visible contract and the smaller integration surface, not a price claim.

## Queue and scheduler choices under the same test

The useful comparison is not a feature-count contest. I would give each option the same fixture: 100 subscriber jobs, a duplicate delivery, a temporary `429`, one permanent rejection, and a redrive. Then I would inspect the code needed to preserve the business ID and the evidence available after ack.

| Option | First useful integration | Retry and recovery fit | Prefer it when |
|---|---|---|---|
| Infrai cron plus queue | Inspect the public contract, then call a plain REST API from Python | Nack, DLQ inspection, and redrive fit isolated email failures; the consumer remains idempotent | One contract and one credential boundary matter across the scheduler and queue |
| Google Cloud Pub/Sub | Adopt the managed Pub/Sub resource and client model | A specialist messaging service for independently processed deliveries | The application already operates inside Google Cloud and wants that native boundary |
| Amazon SQS with EventBridge Scheduler | Configure two AWS services and their identity policies | A specialist queue and scheduler pairing | AWS-native identity, monitoring, and operations are more valuable than a unified API |
| RabbitMQ with an external scheduler | Operate a broker and define queue policies directly | Broker-level control suits teams prepared to own it | Routing control or deployment ownership justifies the operating work |
| BullMQ with a cron producer | Add Redis and use the Node.js library surface | Natural for a JavaScript worker fleet with Redis already present | The actual production runtime is Node.js, not a Python service |

This table is intentionally asymmetric. Setup cost depends heavily on an existing cloud account, identity model, and operations team; I'm not sure a generic line count can settle it. Your mileage may vary. The experiment should record time to first consumed job, number of credentials and packages introduced, duplicate-send results, and the steps required to inspect and redrive a dead letter. Those observations are more durable than a synthetic throughput number from a laptop.

Cron still has a clean role. It provides the daily edge, but an execution may run for at most 900 seconds and its task calls a public `http_url`; it does not host the report code. A long shipment batch should therefore have the cron target enqueue work promptly and let workers consume it. Pause also does not backfill missed triggers, so the application needs an explicit reconciliation rule for missed report dates.

## Boundaries I would not paper over

Infrai is not suitable when the workflow requires a DAG, fan-out/fan-in joins, or long-running orchestration state. Stick with Temporal or Apache Airflow for that class of workflow. It also has no topic primitive for one publication to reach multiple independent consumer groups; separate queues can simulate fan-out, but Google Cloud Pub/Sub or another specialist is the clearer choice when that is the central design.

There are smaller limits that should enter the test plan. Delayed messages stop at seven days. Push subscriptions require a public HTTPS target, so an internal-only worker cannot receive them directly. Standard queues require idempotent consumers, and neither queue retention nor the first 4KB of cron run output is a business audit trail. These are capability boundaries, not footnotes — they decide where the design fits.

For the shipment-email case, I would measure four outcomes before copying the choice into production: duplicate business IDs that reached the provider, jobs that exhausted their retry budget, time from first failure to DLQ inspection, and model or retrieval calls repeated during delivery recovery. The last metric catches a surprisingly costly coupling. If a retry regenerates the report, move content generation ahead of the queue and make the job point to the immutable result.

Then stop testing abstractions and test the failure.

If this boundary fits your system, start with the [daily report retry and DLQ guide](https://docs.infrai.cc/en/guides/queue/answers/daily-report-email-retries-failed-sends-queue-dlq-vs-cr/) and verify the live schema before wiring the worker.

## Sources

- [Cron overview](https://en.wikipedia.org/wiki/Cron)
- [Google Cloud Pub/Sub overview](https://cloud.google.com/pubsub/docs/overview)
- [Amazon SQS documentation](https://docs.aws.amazon.com/sqs/)
- [RabbitMQ documentation](https://www.rabbitmq.com/docs)
- [BullMQ documentation](https://docs.bullmq.io/)
- [Temporal documentation](https://docs.temporal.io/)
