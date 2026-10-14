# Gaming Attribution Webhook Verification with Shared Secrets and Custom Headers

A leaked-key drill is successful only if forged game-usage events cannot acquire a billable identity. **Use a shared-secret HMAC signature over the exact request bytes as the primary webhook check.** Treat a custom header as routing metadata and an IP allowlist as an optional perimeter filter, because neither proves that the sender possessed a signing secret or that the billed payload arrived unchanged.

TL;DR: reject an invalid signature before parsing or attributing usage; then enforce freshness and event uniqueness before writing the billing ledger. During rotation, accept the current and previous signing keys for a bounded overlap, record which key verified the event, and test that a leaked or retired key cannot produce accepted usage. This ordering makes the drill answer the question that matters: did authenticated input, rather than a convenient network signal, cause the charge?

## Why isn't a custom header or IP allowlist enough?

A static custom header is still a secret, but it usually authenticates only the presence of one value. Unless that value is cryptographically bound to the body, an intermediary or application path that can alter the payload can change player, studio, quantity, or event identifiers without invalidating the header. Logging the header also turns ordinary observability plumbing into part of the secret boundary. OWASP's secrets-management guidance recommends limiting secret exposure, auditing access, and supporting rotation; a header copied through several systems makes each control harder to reason about.

An IP allowlist answers a different question: did the connection appear to originate from an expected network address? Proxies, egress changes, shared infrastructure, and address rotation make that useful as a coarse filter but weak as billing evidence. Preserve it as defense in depth when the sender publishes stable ranges and the actual client address can be derived through a trusted proxy chain. Do not let it assign ownership to an event.

| Check | What it establishes | Role in the decision | Failure to rehearse |
|---|---|---|---|
| HMAC over raw body | Possession of a secret and body integrity | Primary authentication gate | Leaked key signs a modified usage event |
| Static custom header | Possession of an unbound value | Routing or secondary signal | Header is exposed in logs or copied to another route |
| Source IP allowlist | Expected network origin | Early perimeter filter | Proxy trust or address changes reject legitimate traffic |

This is why the simple approach fails. Checking a static header and immediately parsing JSON feels efficient in a notebook, yet it does not bind the claimed match minutes to the bytes being billed. The chosen design verifies first and attributes later.

That order is the control.

## The verification boundary

Keep the raw body until authentication finishes. Parsing and reserializing JSON can change whitespace, key order, or numeric representation, so a verifier must calculate its MAC over the same byte sequence the sender signed. Compare digests with a constant-time function. Reject malformed encodings, unknown key identifiers, stale timestamps, and duplicate event identifiers without entering the billing path.

A key identifier is not authentication. It selects a candidate secret from a narrowly scoped key ring; the signature still supplies the proof. Store signing keys outside source code, restrict who and what can read them, audit secret operations, and make rotation a practiced operation. Those controls follow the lifecycle emphasis in the OWASP Secrets Management Cheat Sheet.

For a gaming workload, the authenticated envelope needs stable context: an event ID, studio or account ID, game ID, usage quantity, and signing time. Authentication and authorization remain separate. A valid studio signature does not grant permission to bill another studio's account, so compare the authenticated principal with the payload owner before committing a ledger entry.

## A focused verifier for the drill

This Python example is deliberately framework-neutral. It accepts bytes supplied by the web framework, requires a versioned signature format, places a five-minute ceiling on replay age, and returns the key ID used for verification so the audit record can distinguish old-key traffic during rotation. The secret lookup is an interface boundary; its implementation belongs in the application's secret manager, not in this function.

```python
import hashlib
import hmac
import time
from dataclasses import dataclass
from typing import Mapping


@dataclass(frozen=True)
class VerifiedEvent:
    key_id: str
    received_at: int


class VerificationError(Exception):
    pass


def verify_event(
    raw_body: bytes,
    headers: Mapping[str, str],
    active_keys: Mapping[str, bytes],
    *,
    now: int | None = None,
    max_age_seconds: int = 300,
) -> VerifiedEvent:
    try:
        key_id = headers["x-signature-key-id"]
        timestamp_text = headers["x-signature-timestamp"]
        supplied = headers["x-signature"]
        timestamp = int(timestamp_text)
        version, supplied_hex = supplied.split("=", 1)
    except (KeyError, ValueError) as exc:
        raise VerificationError("malformed signature metadata") from exc

    if version != "v1":
        raise VerificationError("unsupported signature version")

    current_time = int(time.time()) if now is None else now
    if timestamp > current_time + 30 or current_time - timestamp > max_age_seconds:
        raise VerificationError("event timestamp is outside the acceptance window")

    secret = active_keys.get(key_id)
    if secret is None:
        raise VerificationError("unknown signing key")

    signed = timestamp_text.encode("ascii") + b"." + raw_body
    expected_hex = hmac.new(secret, signed, hashlib.sha256).hexdigest()
    if not hmac.compare_digest(expected_hex, supplied_hex):
        raise VerificationError("invalid signature")

    return VerifiedEvent(key_id=key_id, received_at=current_time)
```

This verifier is only the authentication core. After it returns, parse the JSON under a schema with bounds on quantity and identifiers, confirm that the signing principal owns the stated studio account, and insert the event ID through a uniqueness constraint in the same transactional path as the ledger mutation. The timestamp limits delayed replay; the uniqueness constraint handles repeated delivery inside the valid window. Short code. Long boundary.

The exact header names and signed-message format are a protocol contract, not universal standards. Publish that contract to both sides, include test vectors, and version it before changing canonicalization or algorithms. A signature implementation that sender and receiver interpret differently is secure only on paper.

HMAC has a real limitation: both parties must provision, protect, rotate, and eventually revoke the same secret. It is a poor fit when many independent senders cannot be trusted with symmetric key material, when a receiver must verify events without gaining the ability to forge them, or when offline third-party verification is required. An asymmetric signature protocol is the better design in those cases. The trade-off is more key-distribution machinery and usually a more involved protocol contract. Custom headers and IP allowlists do not remove that limitation; they answer narrower questions and should not be promoted into substitutes for payload authentication.

## Running the leaked-key exercise end to end

Start the drill with a dedicated test account and synthetic game-usage events that can never enter production settlement. Capture a baseline: one correctly signed event should create one attributed ledger record, while an identical redelivery should create none. Then expose the previous test signing key to the exercise actor through the same controlled channel modeled by the scenario. Do not place a real production secret in a ticket, chat transcript, fixture, or command history.

Rotate the key. During a predeclared overlap, both current and previous keys may verify, but every accepted event must record its key ID. End the overlap by removing the previous key from the active set, then submit four probes: a valid current-key event, an old-key event, a modified body carrying an unchanged signature, and a replay of the valid event ID. The expected billing effects are one, zero, zero, and zero.

Run them in that order. The first probe proves the rotation did not break current traffic. The second proves revocation reached the verifier. The third isolates body integrity from key selection, since it retains plausible metadata while changing billable content. The fourth moves beyond authentication and checks the ledger boundary itself. For every probe, capture the authentication decision, selected key ID, authorization decision, idempotency result, and final count of ledger rows under one correlation ID. Then query the ledger independently instead of trusting the handler response. A rejected request followed by a write from an already queued task is still a failed drill, and it is exactly the kind of split-path bug that status-code-only checks miss.

That final assertion matters more than a status-code screenshot. Authentication failures should produce no attribution record and no downstream usage side effect, while operational telemetry retains a reason category without retaining the secret or full signature. Alert on rejected signatures, unknown key IDs, replay attempts, and unusual old-key volume during overlap. Keep response details bland enough that they do not become an oracle for attackers.

Recovery also needs a stopping rule. If events signed by the retired key still arrive after the overlap, investigate the sender deployment or possible compromise; do not quietly extend acceptance until the metric looks comfortable. Availability and attribution accuracy pull in opposite directions here, and billing integrity should decide the test account's result.

## What to measure before adopting this pattern

Measure the verifier as a billing control, not as middleware. The primary metric is false attribution: rejected or unauthenticated events that nevertheless cause a ledger write must remain zero in the exercise. Pair it with duplicate ledger writes, valid-event rejection during rotation, verification latency, unknown-key volume, and the time from rotation start until retired-key traffic reaches zero.

An eval harness should replay a fixed corpus across every release: valid signatures, one-byte body changes, malformed hex, missing metadata, timestamps just inside and outside the window, unknown key IDs, duplicate event IDs, and cross-account payloads. Keep expected ledger mutations beside each case. This catches the expensive class of regression where an HTTP handler returns the expected status but attribution has already happened.

Prompt or model usage belongs downstream of this gate. Never spend tokens classifying, enriching, or summarizing an event that has not passed authentication, freshness, deduplication, schema, and account-ownership checks. That ordering reduces wasted inference and prevents untrusted input from entering an agent workflow.

Five minutes is not magic. The example's 300-second window is a starting constraint that must be tested against measured clock skew and delivery delay; widening it improves tolerance for late delivery while increasing the replay interval. Likewise, the 30-second future allowance should be explicit in test vectors rather than buried as an unexplained constant.

No single setting wins.

Copy the pattern only after the sender can sign exact bytes, both parties can rotate keys without an open-ended grace period, and the receiver can enforce idempotency at the ledger boundary. An IP filter may reduce noise. A custom header may simplify routing. The signed body remains the evidence that should control billing attribution.

## Sources

References:

- https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html
