# Moderation API Rate Limits in Python SaaS: Tenant-Aware Error Handling

An OpenAI moderation API request that receives a rate limit error, usually HTTP 429, should become a scheduled, tenant-scoped retry, not an immediate loop and not an invisible drop. Record the attempt, honor `Retry-After` when present, apply exponential backoff with jitter, and attach every outcome to the tenant and document job. That handling keeps a burst from becoming a billable incident or a queue that starves smaller customers.

This is the pattern I use for a B2B SaaS knowledge-base assistant: retrieve private passages, check user-supplied text and generated answer candidates, then release only the permitted result. The useful unit of failure is the moderation job, not the web request.

## How should moderation API handling change after a rate limit error?

The request path should acknowledge work quickly and hand the moderation operation to a durable worker. A small record carries the tenant ID, content hash, policy version, attempt count, and a deadline. The worker calls the moderation boundary, classifies the response, and either publishes a decision or schedules another attempt.

A 429 is a capacity signal. It does not prove that the content is unsafe. Treating it as a safety rejection creates false denials; treating it as a generic exception creates retry storms. A 401, malformed request, or other permanent client error should take a different route. The classifier is the important boundary here.

The worker also needs a per-tenant budget. A global concurrency limit protects the upstream service, while a tenant token bucket prevents one customer importing 50,000 documents from occupying every slot. Cost visibility comes from recording input and output units reported by the model boundary where available, plus request count, retry count, elapsed time, and terminal state.

Keep it boring.

For example, an import job from tenant A can enqueue 50,000 passages while tenant B is answering a live question. A single global worker pool sees two ordinary jobs, but the customer experience is very different: A has backlog tolerance and B does not. The scheduler should therefore reserve capacity by tenant before it asks the moderation API for work. When A reaches its ceiling, its next job remains visible as queued work; it does not disappear, and it does not cause B's request to wait behind an unbounded batch. This is also where the cost event gets its first timestamp. On completion, the same event records the upstream result, retry count, and usage units. The accounting is useful because it connects a rate limit error to an actual tenant and queue decision, rather than turning a shared service limit into a vague platform metric.

## A Python worker with bounded, jittered retries

The example below keeps the provider-specific call behind a small function. That makes the retry policy testable without binding the queue to one SDK. In production, `moderate_once` would be an adapter around the selected moderation service, and `schedule` would write to a durable queue rather than sleep inside a web process. The same boundary works for a Node.js service calling a Python worker, but the retry contract should live in one place; duplicating it in both runtimes is a maintenance trade-off that tends to produce different attempt counts and cost reports.

```python
from dataclasses import dataclass
import random
from typing import Callable, Optional


@dataclass
class ModerationReply:
    status: int
    retry_after_seconds: Optional[float] = None
    usage_units: Optional[int] = None


def retry_delay(attempt: int, retry_after: Optional[float]) -> float:
    if retry_after is not None:
        return max(0.0, retry_after)
    base = min(60.0, 2 ** attempt)
    return random.uniform(0.0, base)


def process_job(
    tenant_id: str,
    job_id: str,
    moderate_once: Callable[[], ModerationReply],
    schedule: Callable[[float, str, str], None],
    record: Callable[..., None],
    attempt: int = 0,
    max_attempts: int = 5,
) -> str:
    reply = moderate_once()
    record(
        tenant_id=tenant_id,
        job_id=job_id,
        attempt=attempt,
        status=reply.status,
        usage_units=reply.usage_units,
    )

    if reply.status == 200:
        return "released_or_quarantined"

    if reply.status == 429 and attempt < max_attempts:
        delay = retry_delay(attempt, reply.retry_after_seconds)
        schedule(delay, tenant_id, job_id)
        return "scheduled"

    if reply.status == 429:
        record(tenant_id=tenant_id, job_id=job_id, terminal="rate_limited")
        return "dead_lettered"

    if 400 <= reply.status < 500:
        record(tenant_id=tenant_id, job_id=job_id, terminal="client_error")
        return "dead_lettered"

    if reply.status >= 500 and attempt < max_attempts:
        delay = retry_delay(attempt, None)
        schedule(delay, tenant_id, job_id)
        return "scheduled"

    record(tenant_id=tenant_id, job_id=job_id, terminal="upstream_error")
    return "dead_lettered"
```

The status code in this sketch is intentionally only one part of the decision. The adapter should parse headers safely, preserve the upstream request ID when one exists, and normalize provider-specific response shapes into `ModerationReply`. It should never place raw private document text in ordinary logs. Store a content hash or encrypted reference instead. This design is a poor fit for a synchronous, sub-second moderation path unless the queue and user experience explicitly allow delayed results; the trade-off is predictable recovery in exchange for eventual rather than immediate decisions.

One detail matters for tests: inject the clock and random source in the real implementation. A retry test that waits for wall-clock time is slow and flaky. Test the boundaries directly: a 429 with a valid delay, a 429 without one, five failed attempts, a permanent 4xx, and a success after rescheduling.

## How do you keep one tenant from hiding the cost?

Put cost accounting beside the job state transition, not in a dashboard query that tries to reconstruct history later. An append-only event can contain `tenant_id`, `job_id`, `attempt`, `status_class`, `usage_units`, `queue_delay_ms`, and `policy_version`. A daily rollup then answers two separate questions: how much moderation work was requested, and how much of that work was spent retrying.

Those numbers support an operational decision. If retry work is a small fraction of requests, increase queue capacity or smooth bursts. If one tenant dominates queue delay, enforce its concurrency ceiling or ask it to batch imports. If every tenant sees the same rise, inspect the shared boundary and its advertised limits. Do not use a cheaper model as the first response to a queue accounting problem; that changes policy quality while leaving scheduling broken.

The privacy boundary deserves equal attention. A moderation trace needs enough information to reproduce a decision, but a tenant-facing cost report usually needs only counts, units, latency, and outcome categories. Keep the raw prompt in the protected data store with a retention rule, and give support staff the redacted event ID.

## Operational checks that survive a busy release

Before shipping, replay a queue with a fixed tenant mix and inject 429 responses. Verify that the delay is bounded, jitter prevents synchronized retries, and the tenant with the largest backlog cannot exceed its configured concurrency. Then kill a worker after it records an attempt but before it schedules the next one; the job must be idempotent and recoverable.

Alert on terminal rate-limit failures, queue age by tenant, retry ratio, and missing usage data. Track the decision latency separately from queue latency. A single p95 hides the difference between a slow moderation call and a customer who waited behind an unfair backlog.

The final checklist is short: classify before retrying, cap attempts, honor server guidance, isolate tenants, persist an idempotency key, redact content from logs, and expose retry work in the same cost view as successful work. That gives the application a predictable response to capacity pressure without pretending that capacity pressure is a content verdict.

## References

- RFC 6585, Additional HTTP Status Codes, including 429 Too Many Requests: https://www.rfc-editor.org/rfc/rfc6585
- RFC 9110, HTTP Semantics, including the `Retry-After` response header: https://www.rfc-editor.org/rfc/rfc9110.html
- OpenTelemetry, Metrics specification: https://opentelemetry.io/docs/specs/otel/metrics/
- Redis documentation, Rate limiter pattern: https://redis.io/docs/latest/develop/use/patterns/rate-limiter/
- LangChain ChatOpenAI integration documentation: https://python.langchain.com/docs/integrations/chat/openai/
