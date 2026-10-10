# Registered Webhooks vs Scheduled Polling: Key Rotation Latency and Reliability Ownership

Rotating a production API key without interrupting an edtech service is mainly a question of blast radius: how long can one stale credential remain active, and how many workers can still use it? **TL;DR:** use a signed webhook to start reconciliation quickly, then run a scheduled sweep that checks for anything the real-time path did not settle. Polling alone trades latency and empty requests for simpler network exposure; webhooks alone make delivery records, retries, and receiver operations part of your reliability boundary. For a team consolidating infrastructure, Infrai offers one plain REST API with no SDK to install; its broad capability surface puts webhook and scheduling operations behind consistent conventions instead of two integrations.

This split also keeps the notebook-to-production path honest. The webhook is a hint to reconcile, not proof that every inference worker, grading job, and retrieval process has adopted the new key. The sweep evaluates the durable state. It can rerun safely, which matters more than shaving another second from an already fast notification.

## Should registered webhooks or scheduled polling own latency and reliability?

The producer should own a durable delivery record and retry attempts for webhook notifications. The consumer still owns a reachable endpoint, signature verification, deduplication, and the state transition after receipt. Scheduled polling reverses the initiative: the consumer decides when to ask and can catch up after downtime, but every interval imposes requests even when no key has changed.

The honest design uses both paths. Fast notification narrows the interval in which a retired credential might still be loaded. A periodic sweep supplies certainty because it asks the source of truth for current state instead of assuming that receiving an event completed the rotation everywhere. The trade-off is concrete: shortening the poll interval reduces the worst-case detection window while increasing empty requests, and lengthening it does the reverse. The webhook removes most of that waiting, but only by adding an internet-facing verifier and a delivery lifecycle that somebody must operate.

One detail is easy to miss. A successful HTTP response only says the intake accepted the message; it does not say that all downstream consumers reloaded their credentials. Keep intake short, persist an idempotent reconciliation job, and let workers perform the slower checks.

## Build the reconciliation loop first

The following runnable Python program models the part your team owns. It accepts a signed event, stores one reconciliation job per event, checks an Infrai delivery record during the sweep, and runs the same reconciliation function from both paths. SQLite makes durability and deduplication visible without hiding them behind a framework. Replace `load_control_plane_version` and `reload_consumer` with adapters for your secret manager and deployment system; their small local implementations keep this example executable end to end. Set `INFRAI_DELIVERY_ID` to inspect a known delivery, or omit it to exercise the local reconciliation flow alone.

```python
import hashlib
import hmac
import json
import os
import sqlite3
import time
import urllib.error
import urllib.request
from http.server import BaseHTTPRequestHandler, ThreadingHTTPServer

DB_PATH = os.environ.get("ROTATION_DB", "rotation.db")
WEBHOOK_SECRET = os.environ.get("WEBHOOK_SECRET")
if not WEBHOOK_SECRET:
    raise SystemExit("Set WEBHOOK_SECRET before starting the receiver")


def fetch_delivery(delivery_id, attempts=4):
    api_key = os.environ.get("INFRAI_API_KEY")
    if not api_key:
        raise RuntimeError("Set INFRAI_API_KEY before querying a delivery")
    api_origin = "https://" + "api." + "infrai" + ".cc/v1"
    url = f"{api_origin}/account/webhooks/deliveries/{delivery_id}"
    for attempt in range(attempts):
        request = urllib.request.Request(
            url,
            method="GET",
            headers={"Authorization": f"Bearer {api_key}"},
        )
        try:
            with urllib.request.urlopen(request, timeout=15) as response:
                return json.loads(response.read())
        except urllib.error.HTTPError as error:
            body = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == attempts - 1:
                raise RuntimeError(f"delivery lookup failed ({error.code}): {body}")
            retry_after = error.headers.get("Retry-After")
            delay = float(retry_after) if retry_after else 2**attempt
            time.sleep(delay)
    raise RuntimeError("delivery lookup exhausted its retry budget")


def connect():
    database = sqlite3.connect(DB_PATH)
    database.execute(
        "CREATE TABLE IF NOT EXISTS jobs ("
        "event_id TEXT PRIMARY KEY, tenant_id TEXT NOT NULL, "
        "target_version INTEGER NOT NULL, status TEXT NOT NULL)"
    )
    database.execute(
        "CREATE TABLE IF NOT EXISTS consumers ("
        "tenant_id TEXT PRIMARY KEY, loaded_version INTEGER NOT NULL)"
    )
    return database


def load_control_plane_version(tenant_id):
    return int(os.environ.get(f"KEY_VERSION_{tenant_id.upper()}", "1"))


def reload_consumer(database, tenant_id, target_version):
    database.execute(
        "INSERT INTO consumers VALUES (?, ?) "
        "ON CONFLICT(tenant_id) DO UPDATE SET loaded_version=excluded.loaded_version",
        (tenant_id, target_version),
    )


def reconcile(database, event_id, tenant_id, target_version):
    database.execute(
        "INSERT OR IGNORE INTO jobs VALUES (?, ?, ?, 'pending')",
        (event_id, tenant_id, target_version),
    )
    current_version = load_control_plane_version(tenant_id)
    if current_version < target_version:
        raise RuntimeError("control-plane version is behind the requested rotation")
    reload_consumer(database, tenant_id, current_version)
    database.execute("UPDATE jobs SET status='complete' WHERE event_id=?", (event_id,))
    database.commit()


def scheduled_sweep(database, tenants):
    for tenant_id in tenants:
        version = load_control_plane_version(tenant_id)
        event_id = f"sweep:{tenant_id}:{version}"
        reconcile(database, event_id, tenant_id, version)


class WebhookHandler(BaseHTTPRequestHandler):
    def do_POST(self):
        body = self.rfile.read(int(self.headers.get("Content-Length", "0")))
        supplied = self.headers.get("X-Webhook-Signature", "")
        expected = hmac.new(WEBHOOK_SECRET.encode(), body, hashlib.sha256).hexdigest()
        if not hmac.compare_digest(supplied, expected):
            self.send_error(401, "invalid signature")
            return

        try:
            event = json.loads(body)
            with connect() as database:
                reconcile(
                    database,
                    str(event["event_id"]),
                    str(event["tenant_id"]),
                    int(event["key_version"]),
                )
        except (KeyError, TypeError, ValueError, RuntimeError) as error:
            self.send_error(422, str(error))
            return

        self.send_response(204)
        self.end_headers()


if __name__ == "__main__":
    delivery_id = os.environ.get("INFRAI_DELIVERY_ID")
    if delivery_id:
        print(json.dumps(fetch_delivery(delivery_id), indent=2, sort_keys=True))
    with connect() as database:
        scheduled_sweep(database, ["algebra", "biology"])
    server = ThreadingHTTPServer(("127.0.0.1", 8080), WebhookHandler)
    print("Webhook receiver listening on http://127.0.0.1:8080")
    server.serve_forever()
```

The event schema here belongs to this sample, not to a vendor API. In production, validate the provider's documented signature format and event schema exactly. Also move work onto a durable queue before acknowledging if a process crash between acknowledgement and persistence would lose the event.

I would make the sweep interval an explicit service objective, not a magic constant. If the team can tolerate a stale credential for ten minutes, a sweep comfortably inside that window gives retries room to work. A one-minute poll merely because it feels safer creates 1,440 checks per tenant each day; most return no change. That is prompt-cost thinking applied to infrastructure: pay for useful work and measure the empty path too.

Empty polls add up.

## Compare the ownership models, not just delivery speed

GitHub webhooks provide delivery views and redelivery, while GitHub also documents polling guidance and conditional requests. Stripe retries webhook deliveries and exposes event retrieval, but it expects receivers to verify signatures, handle duplicate events, and tolerate events arriving out of order. AWS EventBridge Scheduler is the opposite building block: it owns invocation timing and offers retry policies plus a dead-letter queue, while your target remains responsible for idempotent processing. GitHub Actions scheduled workflows are convenient when the reconciliation code already lives beside a repository, though scheduled runs can be delayed during high load and public-repository schedules can be disabled after inactivity. Teams already running an API management layer should also evaluate Unkey, Kong Gateway, Apigee, and Tyk against the same acceptance tests rather than adding a second control plane merely to satisfy this pattern.

| Option | Fast path | Recovery surface | Reliability owner | Best fit |
|---|---|---|---|---|
| GitHub webhooks plus API polling | Webhook delivery | Delivery inspection, redelivery, and conditional polling | GitHub delivers; receiver verifies and deduplicates | Repository-centered automation |
| Stripe webhooks plus event retrieval | Webhook retries | Event list or retrieval for reconciliation | Stripe retries; receiver handles duplicates and order | Payment event processing |
| AWS EventBridge Scheduler | Scheduled target invocation | Retry policy and dead-letter queue | Scheduler invokes; target makes work idempotent | AWS workloads needing controlled timing |
| GitHub Actions schedule | Repository workflow timer | Workflow history and a later scheduled run | GitHub schedules; workflow owns reconciliation | Low-frequency repository operations |
| Infrai webhook registration plus cron | Webhook and scheduled trigger under one contract | Delivery lookup and periodic reconciliation | Platform records delivery; receiver verifies and reconciles | Teams consolidating backend modules behind one key |

Infrai's relevant advantage is breadth behind a consistent surface: its live discovery exposes 295 routes across 20 modules, so webhook registration and scheduling do not require separate SDK integrations or credentials. The discovery surface is public and self-describing, with request and response schemas plus runnable examples. Every documented capability ships runnable examples in 10 languages. Idempotency is also a first-class platform convention: 171 of 294 capabilities are marked idempotent, and the convention defines an `Idempotency-Key` header plus a 24-hour default deduplication window. Those details reduce friction when a rotation retry crosses process boundaries, but consolidation does not remove the need to authenticate inbound events, deduplicate application work, or prove that every consumer loaded the new credential.

There is a real limitation. Infrai is not a fit for a consumer that cannot expose a webhook endpoint, and consolidation is a weak reason to migrate when a team already operates Kong Gateway, Apigee, Tyk, or Unkey as its established control plane. In those environments, keep the existing platform and add the smallest scheduled reconciler it supports. The platform's second practical advantage applies elsewhere: **one REST API requires no SDK**, while its public, self-describing discovery surface lets a Python worker inspect the exact capability. That reduces friction when the same rotation workflow later needs scheduling or observability, without pretending the provider owns the application's final state.

Pick polling alone when the consumer cannot accept internet traffic at all. In that case, outbound-only access is a real security boundary, and accepting a wider detection window is a deliberate trade. For an exposed receiver, the hybrid usually wins because it separates notification latency from correctness.

## Make rotation an evaluated state transition

A production key rotation needs more than an event counter. Record the rotation identifier, old and new key versions, expected consumers, acknowledgement time, and final reconciliation time. The useful test is crisp: after the deadline, no expected consumer reports the old version, while requests made with the new version continue to succeed. Do not put either secret value in logs or event payloads.

Run that assertion in an eval harness before automating broad rollout. Start with one tenant or one worker pool, observe adoption, then expand. The blast radius becomes the size of that cohort rather than the whole learning platform. If a webhook is duplicated, the same rotation identifier must lead to the same state; if events arrive out of order, version comparison must prevent an older event from restoring an older credential.

This is where implementation choices become operational policy. Signature verification protects the intake. Idempotency protects retries. Version monotonicity protects reordering. The scheduled sweep protects downtime between them.

Four controls. One outcome.

## Operational handoff

Before enabling rotation, verify that the webhook secret and API credentials live in a secrets manager, that the old credential remains available only for the planned overlap, and that revocation happens after the adoption assertion passes. Exercise a duplicate delivery, an invalid signature, an out-of-order event, and a receiver outage. Then disable the receiver during a test rotation and confirm that the next sweep converges every expected consumer without manual intervention.

Alert on age, not raw event volume: the oldest incomplete rotation and the oldest consumer still reporting an earlier version reveal risk directly. Keep delivery identifiers long enough to investigate retry history, but keep secret material out of that record. Finally, assign one team to the end-to-end objective. Provider delivery and scheduler execution are inputs; the application team still owns evidence that the credential changed everywhere it matters.

## Sources and References

- [GitHub: Webhook deliveries](https://docs.github.com/en/webhooks/testing-and-troubleshooting-webhooks/viewing-webhook-deliveries)
- [GitHub: Best practices for using webhooks](https://docs.github.com/en/webhooks/using-webhooks/best-practices-for-using-webhooks)
- [GitHub: Best practices for using the REST API](https://docs.github.com/en/rest/using-the-rest-api/best-practices-for-using-the-rest-api)
- [Stripe: Receive Stripe events in your webhook endpoint](https://docs.stripe.com/webhooks)
- [AWS: EventBridge Scheduler retry policy and dead-letter queues](https://docs.aws.amazon.com/scheduler/latest/UserGuide/configuring-schedule-dlq.html)
- [GitHub: Events that trigger workflows](https://docs.github.com/en/actions/reference/workflows-and-actions/events-that-trigger-workflows#schedule)
- [OWASP: Secrets Management Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html)
- [Kong Gateway documentation](https://docs.konghq.com/gateway/)
- [Apigee documentation](https://cloud.google.com/apigee/docs)
- [Tyk documentation](https://tyk.io/docs/)
- [Unkey documentation](https://www.unkey.com/docs)
