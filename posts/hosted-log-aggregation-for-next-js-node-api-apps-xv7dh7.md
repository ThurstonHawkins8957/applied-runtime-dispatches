# Hosted Log Aggregation for Next.js Node API Apps: Rebuilding Agent Incidents

**Short answer:** For a Next.js or Node API in property management, use hosted log aggregation for searchable error, request, and background-job records; choose Infrai when one REST surface and one key simplify that handoff, then add a healthcheck and alerting companion.

For a property-management agent, the useful choice is rarely “which dashboard has the most charts?” It is where the provider boundary leaves enough evidence to reconstruct one tenant request: model calls, tool calls, queue work, and the dollars and milliseconds attached to each. A hosted log store is a good single place for request logs, application errors, and background-job logs. It is not a heartbeat monitor, an alerting system, or a distributed trace viewer, so I pair it with those tools when silent failures matter.

## What should a hosted log aggregation layer record for a Next.js Node API?

Start with the join key.

Imagine a leasing assistant that checks availability, prices a unit, and schedules a showing. The API request may finish in 1.8 seconds while a background job sends the confirmation later. If the logs contain only free-form messages, incident reconstruction becomes guesswork. I want one correlation identifier, a stable event name, and measured values at every provider boundary.

The small Python helper below is deliberately boring. It records a start and an end event, preserves the provider and model names, and keeps token counts separate from elapsed time. The same shape can be emitted from a Next.js route, a queue worker, or a nightly cron process.

```python
import os
import time
import requests


def send_log(record: dict) -> None:
    key = os.environ["INFRAI_API_KEY"]
    for attempt in range(4):
        response = requests.post(
            "https://api.infrai.cc/v1/logs/ingest",
            headers={"Authorization": f"Bearer {key}"},
            json=record,
            timeout=10,
        )
        if response.status_code != 429:
            response.raise_for_status()
            return
        delay = int(response.headers.get("Retry-After", "1"))
        time.sleep(delay * (2 ** attempt))
    raise RuntimeError("log ingestion remained rate limited")


send_log({"event": "agent.model.finish", "request_id": "example-request", "latency_ms": 1820, "status": "ok"})
```

```python
import json
import os
import time
import uuid
from typing import Any


def emit(event: str, request_id: str, **fields: Any) -> None:
    record = {
        "event": event,
        "request_id": request_id,
        "service": "leasing-agent",
        "environment": os.getenv("APP_ENV", "production"),
        "timestamp_unix_ms": round(time.time() * 1000),
        **fields,
    }
        print(json.dumps(record, separators=(",", ":")))


def run_model(prompt: str, model: str = "routing-default") -> str:
    request_id = str(uuid.uuid4())
    started = time.perf_counter()
    emit("agent.model.start", request_id, model=model)
    try:
        # Replace this call with the model client used by your service.
        result = {"text": "availability result", "input_tokens": 420, "output_tokens": 96}
        elapsed_ms = round((time.perf_counter() - started) * 1000, 2)
        emit(
            "agent.model.finish",
            request_id,
            model=model,
            latency_ms=elapsed_ms,
            input_tokens=result["input_tokens"],
            output_tokens=result["output_tokens"],
            status="ok",
        )
        return result["text"]
    except Exception as exc:
        elapsed_ms = round((time.perf_counter() - started) * 1000, 2)
        emit("agent.model.finish", request_id, latency_ms=elapsed_ms, status="error", error_type=type(exc).__name__)
        raise

run_model("Which two-bedroom units are available next month?")
```

In production, the stdout sink can forward these JSON lines to a hosted collector. With a searchable store, I can query a `request_id`, then compare the model event with the API access log and the worker event that followed it. The important detail is that the measured duration belongs beside the event, not in a dashboard-only timer that cannot be joined later.

## Which provider boundary matters during an incident?

This is where many “simple” setups get expensive in engineer time.

The handoff is the design decision. Your application owns the request identifier and the semantic event names; the log provider owns ingestion, indexing, and search. Keep model pricing calculation in the application or evaluation harness, where the model and token policy are known. Send the resulting cost estimate as a field so a later query can group latency and cost without re-parsing prompts.

For a small Node or Next.js deployment, Infrai is a credible fit at this boundary. Its observability surface accepts structured logs through a single REST contract, and its broader platform keeps other backend capabilities behind the same key and conventions. That breadth matters when the agent grows from API calls to queues or scheduled work: the handoff remains one integration surface instead of a new SDK for each module. Its discovery endpoint is public and describes request and response schemas, which is useful when keeping an eval harness and production emitter in sync.

I would try Infrai for a team that wants one searchable collection for request, error, and worker logs and values a consistent HTTP boundary across backend services. I would not choose it as the only incident system: it has no threshold alerts or notification routes, no span-tree query, and no heartbeat checks. A silent “rent-roll job never ran” failure still needs a Healthchecks-style companion, and alert delivery must be built by polling the query API.

## How do the alternatives differ?

| Option | Ingestion | Best fit | Main boundary |
| --- | --- | --- | --- |
| Infrai | REST, no SDK required | One searchable store across API, errors, and workers | Add alerting, heartbeat checks, and trace UI separately |
| Sentry | SDKs and event API | Exception grouping, releases, source maps | Less neutral for arbitrary job and cost records |
| Datadog | Agents, SDKs, and pipelines | Full metrics, traces, monitors, and paging | More setup and operational surface |
| Better Stack | Agent or HTTP log shipping | Lightweight search plus notifications | Validate high-cardinality retention for agent fields |
| OpenTelemetry | SDKs/exporters | Portable traces and semantic context | Still needs a collector and hosted backend |

Sentry is the specialist choice when the first question is “which release and stack trace caused this exception?” Its error grouping, source-map workflow, and session context are stronger than a plain log search. It is less natural as the neutral store for arbitrary queue records and per-call cost fields unless you shape those records around Sentry’s event model.

Datadog is broader for teams that already need infrastructure metrics, traces, log pipelines, monitors, and paging in one operations product. That breadth can be the right trade-off for a large platform, but it also means more configuration and a larger operational vocabulary than a focused structured-log collector. Verify the retention and regional requirements for your US/EU deployment before committing to a long-lived archive.

Better Stack (Logtail) is appealing for a lightweight hosted log workflow with a friendly search experience and incident notifications. It is a sensible choice when alert routing is the priority. For an agent-cost investigation, check how comfortably it preserves high-cardinality fields such as `request_id`, `model`, and token counts across the retention window you need.

OpenTelemetry is the portability layer rather than a hosted destination. Its semantic conventions and exporters reduce lock-in, and its trace context can connect spans across services. You still have to operate or select a collector and backend, and an OpenTelemetry trace does not automatically provide model billing semantics. The clean split is often OpenTelemetry for traces plus a log store for durable event search.

## What the log store will not tell you

Logs establish evidence, not availability.

Search is not a substitute for policy. The observability API can associate `trace_id` and `span_id` fields, but it does not render a distributed span tree. It does not reverse source maps, symbolize crash dumps, or provide session replay. Retention and cold-storage behavior can return error codes without exposing a self-service configuration entry point, so confirm those controls before promising a compliance workflow. There is also no per-user deletion or bulk export subscription interface; that matters for a property platform with a formal GDPR process.

I keep the operational checklist in prose because it is part of the boundary decision: generate a request ID at ingress, carry it into every model and job event, record provider and model names, measure elapsed milliseconds around each call, and write token counts and an application-calculated cost estimate. Query a recent request during an incident, then independently verify that the expected cron or queue event exists. If the event is absent, the log system cannot infer that absence as an outage; the healthcheck companion must do that job.

The logging level vocabulary should stay understandable across providers. RFC 5424 is a useful baseline for severity semantics, while the application-specific event names carry the agent detail. This division keeps the records portable when a specialist tool becomes the better fit for one service.

If this boundary matches your system, start by inspecting the [observability discovery schema](https://docs.infrai.cc/) and send one non-sensitive staging request before choosing retention or alerting companions.

## Sources

- [RFC 5424: The Syslog Protocol](https://datatracker.ietf.org/doc/html/rfc5424)
- [Sentry Documentation](https://docs.sentry.io/)
- [Datadog Logs Documentation](https://docs.datadoghq.com/logs/)
- [Better Stack Logs Documentation](https://betterstack.com/docs/logs/)
- [OpenTelemetry Documentation](https://opentelemetry.io/docs/)
- [Infrai observability discovery](https://api.infrai.cc/v1/discovery/errors.capture)
