# Property Checkout Safety: Node.js Cron Health Checks and Heartbeats for Missed Jobs

Short answer: for a Node.js cron health check, use a validated completion event, searchable run records, and an outside deadline for missed-job detection. Make rollback safety the acceptance rule. Retries can recover a failed invocation; they cannot prove that the scheduler launched anything. A heartbeat should be sent only after the checkout state is known to be safe, and its timeout must cover the full runtime and retry budget.

That distinction is easy to miss in a property-management system. A green process monitor can coexist with a missed checkout reconciliation, while a retry can repeat a payment or key-release side effect unless the operation is idempotent. The monitor is useful only when its “success” means the property workflow reached a reversible, verified state.

## How can a Node.js cron health check detect a missed job?

Start by defining the state transition rather than choosing a monitoring product. For each scheduled run, record `scheduled`, `started`, `validated`, `rolled_back`, or `failed`. Include a run ID, property ID, checkout batch ID, attempt number, and the reason for any rollback. A process exit code is evidence about the process, not about whether the reservation, inventory, or access-control update is safe to keep.

The outside health check answers a narrower question: did this run report a valid terminal outcome before its deadline? If the job never starts, the application can't report its own absence. If it starts and stalls, the deadline should alert without pretending to know whether a partial checkout update is safe to undo — that decision belongs to the workflow's transaction boundary and reconciliation logic.

Keep the success signal last.

For a checkout batch, validate the output first: compare the expected record count, verify the state transition, and confirm that the rollback marker is durable when the run did not finish safely. Only then send the heartbeat. A failed validation must leave a searchable failure record and no success signal. This makes missed-job detection distinct from ordinary exception monitoring.

## How do timeout, retries, and heartbeat monitoring interact?

Put retries inside one execution window. Suppose the Node.js command normally takes eight minutes and allows two retries with backoff. A ten-minute timeout can report a false missed run while the final attempt is still doing legitimate work. Set the deadline from an observed duration budget: maximum execution time, retry delays, scheduler jitter, and a small delivery allowance. There is no universal value; your mileage may vary with database locks and upstream latency.

Retries also need an idempotency key. Reusing the same run ID across attempts lets the checkout service reject or safely absorb a duplicate operation. Never make “heartbeat delivered” the condition for committing business data. Delivery is an observation about the result, not the result itself.

The two failure drills are different. Force the child command to exit with code `124` and check that the failure record is present without a success ping. Then omit the entire scheduled launch and wait beyond the heartbeat deadline. The first test exercises error handling; the second exercises missing-run detection. Both belong in the deployment acceptance harness.

## A Python supervisor for the rollback-safe scheduled run

The flow below keeps the scheduler, the Node.js worker, and the external health check separate. The endpoint is a generic heartbeat URL supplied through the environment; the monitor may be hosted or self-hosted. The supervisor does not decide whether a checkout is safe. The child command does that validation and exits successfully only after its durable state is correct.

```python
import json
import os
import subprocess
import time
from datetime import datetime, timezone
from urllib.request import Request, urlopen


def emit(event, **fields):
    print(json.dumps({
        "event": event,
        "at": datetime.now(timezone.utc).isoformat(),
        **fields,
    }), flush=True)


def send_heartbeat():
    request = Request(os.environ["HEARTBEAT_URL"], method="POST")
    with urlopen(request, timeout=10) as response:
        if not 200 <= response.status < 300:
            raise RuntimeError(f"heartbeat status: {response.status}")


def run_checkout():
    run_id = os.environ["CRON_RUN_ID"]
    property_id = os.environ["PROPERTY_ID"]
    emit("checkout_started", run_id=run_id, property_id=property_id)

    for attempt in range(1, 4):
        result = subprocess.run(
            ["node", "checkout-reconcile.js", "--run-id", run_id,
             "--property-id", property_id],
            check=False,
            timeout=900,
        )
        if result.returncode == 0:
            emit("checkout_validated", run_id=run_id,
                 property_id=property_id, attempt=attempt)
            send_heartbeat()
            return

        emit("checkout_attempt_failed", run_id=run_id,
             property_id=property_id, attempt=attempt,
             exit_code=result.returncode)
        if attempt < 3:
            time.sleep(2 ** attempt)

    raise SystemExit("checkout reconciliation exhausted its retry budget")


if __name__ == "__main__":
    run_checkout()
```

The important detail is not the HTTP call. It is the ordering. A successful child exit must mean that the worker checked the property workflow and recorded the correct rollback or completion state. In production, send the same run ID to logs and metrics, and keep the payload small enough that an operator can search it during an incident. For an AI-assisted reconciliation step, also record the prompt version, evaluator version, and token counts when available; those fields connect notebook experiments to production behavior without turning the health check into a second application log.

## Which signal owns each checkout failure?

An external heartbeat is a good fit when the primary question is “did the scheduled work report on time?” It is not a substitute for transaction semantics, an audit trail, distributed tracing, or a reconciliation queue. A self-hosted monitor can be attractive when data must remain inside the operating boundary, but it adds ownership for upgrades, storage, alert delivery, and its own availability. A hosted monitor removes some maintenance while adding a dependency and a data-path review.

The catch is operational ownership. If the heartbeat service is unreachable, the checkout job still needs a durable local outcome and an operator-visible record; otherwise a notification problem becomes a data-integrity problem. Stick with an existing telemetry stack when it already owns on-call routing and retention, and choose a queue or transaction coordinator when rollback requires coordinated writes across systems. Choose a dedicated scheduled-job monitor when silence detection is the only missing capability. The right alternative depends on which failure you must prove, not on whether the dashboard looks complete.

| Failure to catch | Evidence that should exist | Signal that should alert |
| --- | --- | --- |
| The cron invocation never happened | No `started` record for the expected run ID | Outside deadline or heartbeat monitor |
| The worker started and failed | `started` plus `failed` records and exit code | Application alert, without a success heartbeat |
| A checkout was partially applied | Validation result and durable rollback marker | Workflow reconciliation owner |
| A retry could duplicate a write | Same idempotency key across attempts | Test failure before deployment |

## Operational acceptance for a missed-run monitor

Before shipping, write down the schedule, maximum runtime, retry count, backoff, heartbeat deadline, and rollback owner in one reviewable place. Test a normal validated checkout, a validation failure, a process timeout, a retry that reuses the same run ID, a completely omitted launch, and an unavailable heartbeat endpoint. For each case, assert the expected state transition and confirm that no unsafe success signal is emitted.

Then check the boring parts: logs are searchable by run ID, metrics distinguish attempts from runs, secrets are injected rather than committed, and the monitor's alert reaches the actual on-call path. Re-run the timing test after workload changes. A small acceptance harness is more valuable than a permanently green check whose success definition nobody can explain.

Three signals. Clear ownership.

## References

- https://web.dev/articles/vitals
- https://clickhouse.com/docs
