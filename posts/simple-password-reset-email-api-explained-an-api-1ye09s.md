# Simple Password Reset Email API Explained (An API-First Implementation)

Short answer: choose an HTTP email API by the evidence it can return and preserve, not by how little code the first send requires. For a fintech application that issues an order receipt after payment settles, password recovery should reuse the same evidence model: record intent before delivery, attach a stable internal message ID, accept later events idempotently, and keep authentication success separate from transport success. No SMTP relay is required for that boundary.

The smallest useful abstraction is not `send_email(to, subject, body)`. It is `submit_message(message_id, template, recipient, evidence_context)`, followed by independently recorded delivery events. An HTTP 2xx response proves that one system accepted a request; it does not prove that a mailbox received a message.

## How should a password reset email API handle first delivery?

A password reset and a settled-payment receipt have different business meanings, but the evidence questions are nearly identical. What did the application intend to send? Which policy and template version produced it? When did the delivery adapter accept it? Which later event changed its state? The answers should survive retries, delayed callbacks, and a provider migration.

I use four internal transport states: `created`, `accepted`, `delivered`, and `failed`. They are deliberately plain. A provider-specific label belongs in the raw event payload, while the normalized state belongs in application data. This keeps an audit query readable without pretending every transport reports identical details.

Keep the authentication record separate. A reset token can be consumed even when a delivery callback is late, and a delivery event must never mark a token as used. Likewise, the settled order ID is evidence context for a receipt, not a transport identifier. Mixing these concepts makes the notebook demo compact and production investigations miserable.

One rule pays for itself quickly: create the local message row before calling the external API. Then a timeout has somewhere to land.

## A minimal runnable boundary

This Python example uses the standard library and a pseudonymous HTTPS endpoint. It computes a recipient fingerprint for operational correlation and records acceptance without calling it delivery. The adapter is small enough for a notebook, while its contract can move behind a worker unchanged.

```python
from __future__ import annotations

import hashlib
import json
import os
import urllib.request
from dataclasses import dataclass
from datetime import datetime, timezone
from typing import Any


@dataclass(frozen=True)
class MessageIntent:
    message_id: str
    recipient: str
    template: str
    template_version: str
    evidence_context: dict[str, str]


def fingerprint(address: str) -> str:
    value = address.strip().lower().encode("utf-8")
    return hashlib.sha256(value).hexdigest()[:16]


def submit_message(intent: MessageIntent) -> dict[str, Any]:
    payload = {
        "message_id": intent.message_id,
        "to": intent.recipient,
        "template": intent.template,
        "template_version": intent.template_version,
        "data": {"reset_url": "https://accounts.example/reset?token=REDACTED"},
        "metadata": intent.evidence_context,
    }
    request = urllib.request.Request(
        os.environ["MAIL_API_URL"],
        data=json.dumps(payload).encode("utf-8"),
        headers={
            "Authorization": f"Bearer {os.environ['MAIL_API_TOKEN']}",
            "Content-Type": "application/json",
            "Idempotency-Key": intent.message_id,
        },
        method="POST",
    )
    with urllib.request.urlopen(request, timeout=5) as response:
        result = json.load(response)
    return {
        "message_id": intent.message_id,
        "recipient_fingerprint": fingerprint(intent.recipient),
        "state": "accepted",
        "accepted_at": datetime.now(timezone.utc).isoformat(),
        "provider_reference": result.get("id"),
    }


intent = MessageIntent(
    message_id="msg_reset_01J9K7M2",
    recipient="customer@example.net",
    template="password_recovery",
    template_version="2026-04-17",
    evidence_context={
        "policy_version": "recovery-v4",
        "request_id": "req_01J9K7KQ",
        "related_receipt": "order_8142",
    },
)
print(json.dumps(submit_message(intent), indent=2))
```

The five-second timeout bounds one attempt; it does not declare the send failed. A timeout leaves the outcome unknown until reconciliation. The token is redacted because transport logs should not become a second credential store. In production, the real short-lived URL reaches the template at the adapter boundary, while structured logs retain only approved evidence fields. The `MAIL_API_URL` value is deployment configuration rather than a route promised by this generic contract; each adapter owns the exact endpoint and response mapping.

Do not copy the sample endpoint or response shape into a domain contract. Map each provider response into a local adapter protocol. The code is short. The state machine is the product.

## Evidence before convenience

I would evaluate candidates with fixed fixtures rather than a feature checklist. One fixture represents a normal recovery request. Another repeats the same `message_id`. A third makes submission time out and later supplies a delivery event. A fourth sends the same callback twice. The pass condition is deterministic local state, not a polished dashboard.

The eval harness should preserve the submitted template version, policy version, request ID, acceptance timestamp, normalized outcome, raw event type, and provider reference. Retention and access rules belong in explicit organizational policy. Store enough to answer the audit question, but avoid retaining a reset URL, token, or full rendered body merely because it was returned.

Email authentication is another gate. SPF, standardized in RFC 7208, lets a domain publish which hosts are authorized to use its name in SMTP `HELO` and `MAIL FROM` identities. An API-first integration does not remove that DNS responsibility; it changes how the application hands work to the delivery system. Verify that the sending-domain arrangement fits the organization's DNS ownership and evidence process.

SMS deserves a separate decision, not an automatic fallback branch. Twilio's public SMS documentation is one example of a programmable messaging interface, but changing channels changes identifiers and the evidence to retain. Treat email and SMS as separate adapters behind one recovery policy. Never interpret an email failure as permission to send a text.

Cost belongs in the harness without becoming the argument. Record attempts per successful recovery, callback volume, payload size, and retained-event growth. Those measurements expose retry storms. Prompt cost is zero here: adding a model to compose a deterministic security message introduces variability without a useful decision to make.

It isn't universal.

This approach is not suitable when the team cannot operate callback ingestion, reconciliation, and evidence retention. In that case, a managed authentication system with recovery delivery included may reduce operational ownership, provided its audit exports satisfy the same test fixtures. Direct SMTP may also be the appropriate choice inside an environment where an established relay already supplies policy enforcement and trace records. The trade-off is control versus owned machinery: an HTTP adapter creates a clean application boundary, but the application still has to model ambiguous outcomes instead of delegating the entire recovery workflow.

## Where does the design fail under retries?

The dangerous case is ambiguous acceptance. The application submits a message, the remote system accepts it, and the connection closes before the response arrives. Retrying with a new ID can create two emails. Declaring failure can suppress a message already moving. Preserve the same internal ID, use the adapter's supported idempotency mechanism when available, and reconcile against later evidence.

Callbacks create a second race. Delivery can arrive before the worker commits `accepted`; duplicate events can arrive apart; a failure can follow an intermediate event. The consumer therefore needs a unique event key, an append-only raw record, and a normalization rule defined by policy. Unknown event types should be retained for review instead of guessed into `failed`.

Return the same neutral response for recovery requests that do and do not match an account, enqueue the work, and let a bounded worker own transport retries. This also gives the receipt flow a clean trigger: payment settlement creates a receipt intent once, while the delivery worker handles network behavior without reopening payment state.

A small implementation can still have sharp boundaries. Good.

## Ship it with an evidence checklist

Before deployment, I run the adapter against the four fixtures and inspect the database, not the inbox alone. The created intent must exist before the network call. A repeated submission must retain one logical message ID. Duplicate callbacks must not duplicate transitions. A timeout must remain unknown until evidence resolves it. Logs must correlate request and message without exposing the reset token.

Then I test the human path: expired link, already-used link, delayed email, and a second request made before the first arrives. The interface should explain the next action without revealing account existence, while the audit trail connects each action to its policy version. For the settled-payment receipt, replaying a delivery event must not alter settlement state.

Choose the API and adapter combination that passes these evidence tests, supports the required sending-domain controls, and leaves an intelligible record during ambiguous failure. Syntax is a weak differentiator; most submission calls fit on a screen. Recovery safety comes from the surrounding state model, and compliance confidence comes from evidence the application can query independently.

## Sources

References:

- RFC 7208, Sender Policy Framework (SPF): https://datatracker.ietf.org/doc/html/rfc7208
- Twilio SMS documentation: https://www.twilio.com/docs/sms
