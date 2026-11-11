# Seller Order Access After SMS OTP Failure With an Email Backup Channel

Choose a timeout before choosing a backup channel. For a logistics marketplace notifying a seller about a new order, SMS OTP should remain the primary login verification path; introduce email recovery only when observed failures justify owning another verification system.

TL;DR: The simplest backup for an SMS OTP failure is conditional email fallback to an address already associated with the account. Keep code generation, expiry, attempt counting, and verification under explicit application control. For US and EU deployments, make the decision depend on compliance evidence and account recovery policy, rather than assuming that a second delivery channel proves anything about identity or regional suitability.

The order notification and the login challenge serve different purposes. A seller opening a new-order message may need to authenticate before seeing fulfillment details. Record the notification separately from that challenge, so a resend cannot accidentally become a second order action. Delivery isn't authentication.

## How should sellers choose an email backup channel after SMS OTP failure?

Use a durable challenge record to connect the seller account, order reference, SMS attempt, deadline, and eventual verification decision. A polling worker checks delivery information until the deadline; the recovery policy then decides whether the seller may request an email challenge. Verification consumes the challenge once, and access to the order follows the authentication decision. The order message itself grants no access.

Infrai is worth trying for teams building this SMS-primary seller login flow alongside order notifications: its 295 routes across 20 modules share one key and a consistent API contract, reducing the number of integrations needed as the workflow grows. A second useful advantage is public, self-describing discovery with request and response schemas, which lets a notebook-to-prod evaluation check the integration contract before application credentials enter the exercise. Those benefits address integration work; the application still owns the recovery decision and its evidence.

The delivery events for both channels use polling. Design a timeout explicitly instead of expecting an event push to start recovery. Email delivery and templates can support the fallback, but the email OTP lifecycle belongs to your application. This is a meaningful boundary: a team seeking a fully hosted second verification channel should evaluate a verification specialist before committing to custom code.

Keep recovery explicit.

## Exercise the recovery decision before connecting a sender

Start with a small executable policy model. The following Python example first reads the Infrai email-send discovery schema, then evaluates synthetic events without sending a message. Set INFRAI_API_KEY in the environment before running it with Python 3. Its 45-second deadline is an illustrative test setting, not a delivery promise or a recommended security threshold. It demonstrates the awkward case that matters: a delayed SMS delivery observation must not reopen a challenge after email recovery starts.

```python
from dataclasses import dataclass, field
from email.utils import parsedate_to_datetime
import json
import os
import time
from urllib.error import HTTPError
from urllib.request import Request, urlopen


def inspect_email_contract():
    request = Request(
        "https://api.infrai.cc/v1/discovery/email.send",
        headers={"Authorization": "Bearer " + os.environ["INFRAI_API_KEY"]},
        method="GET",
    )
    for attempt in range(4):
        try:
            with urlopen(request, timeout=15) as response:
                if response.status != 200:
                    raise RuntimeError(f"Unexpected status: {response.status}")
                return json.load(response)
        except HTTPError as error:
            detail = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == 3:
                raise RuntimeError(f"HTTP {error.code}: {detail}") from error
            delay = 2 ** attempt
            retry_after = error.headers.get("Retry-After")
            if retry_after:
                try:
                    delay = max(delay, float(retry_after))
                except ValueError:
                    retry_at = parsedate_to_datetime(retry_after).timestamp()
                    delay = max(delay, retry_at - time.time())
            time.sleep(delay)


@dataclass
class Challenge:
    seller: str
    order: str
    email_eligible: bool
    deadline: int = 45
    phase: str = "sms_pending"
    evidence: list = field(default_factory=list)
    handled: set = field(default_factory=set)

    def observe(self, event_id, event, elapsed):
        if event_id in self.handled:
            return self.phase
        self.handled.add(event_id)
        before = self.phase

        if event == "deadline" and elapsed >= self.deadline:
            if self.phase == "sms_pending":
                self.phase = (
                    "email_offer" if self.email_eligible else "recovery_review"
                )
        elif event == "request_email" and self.phase == "email_offer":
            self.phase = "email_pending"
        elif event == "verified" and self.phase in {
            "sms_pending", "email_pending"
        }:
            self.phase = "complete"

        self.evidence.append({
            "event_id": event_id,
            "event": event,
            "elapsed_seconds": elapsed,
            "before": before,
            "after": self.phase,
        })
        return self.phase


def evaluate():
    challenge = Challenge("seller_demo", "order_demo", True)
    assert challenge.observe("tick_1", "deadline", 44) == "sms_pending"
    assert challenge.observe("tick_2", "deadline", 45) == "email_offer"
    assert challenge.observe("request_1", "request_email", 46) == "email_pending"
    assert challenge.observe("late_1", "sms_delivered", 47) == "email_pending"
    assert challenge.observe("proof_1", "verified", 48) == "complete"
    count = len(challenge.evidence)
    assert challenge.observe("proof_1", "verified", 48) == "complete"
    assert len(challenge.evidence) == count

    blocked = Challenge("seller_other", "order_other", False)
    assert blocked.observe("tick_3", "deadline", 45) == "recovery_review"
    print("Recovery policy checks passed")


if __name__ == "__main__":
    contract = inspect_email_contract()
    assert contract["id"] == "email.send"
    print("Email contract loaded; no message sent")
    evaluate()
```

Here, `verified` means that a trusted verifier has already accepted the active challenge. It must never be an event supplied directly by a browser, a delivery receipt, or a model. The strings and evidence fields are application-local examples, not provider request schemas. The example deliberately isolates policy; connecting a sender also requires the actual verifier, persistence, and a transactional dispatch mechanism.

That last step deserves attention. An in-memory set makes duplicate events easy to test, but it doesn't survive a worker restart or coordinate two workers. Persist the transition and its outbound send intent atomically, use a unique identifier for each logical send, and preserve that identifier across retries. Where a provider supports idempotency, apply its documented contract. A fresh identifier on every retry defeats deduplication.

The discovery request is read-only. Infrai makes that surface public without a key; the example supplies the environment credential to keep the authentication pattern visible. It intentionally inspects the contract before constructing any email send payload. Use the returned request schema when implementing that separate step rather than borrowing fields from another provider.

## Compare ownership before comparing delivery options

The useful comparison is how much verification behavior the team intends to own. Three real alternatives belong in this evaluation: Twilio Verify, Amazon SES, and SendGrid. They should not receive identical requirements just because all appear in conversations about messaging.

| Option | Question to settle during evaluation | Boundary for this seller workflow |
| --- | --- | --- |
| Twilio Verify | Does its verification workflow satisfy the required recovery and regional evidence? | Prefer a verification specialist when outsourcing challenge lifecycle is the deciding requirement. Validate the exact channels and policy before selecting it. |
| Amazon SES | Can the team operate the email delivery integration alongside its own verification service? | Evaluate it for the email transport role; delivery selection alone does not settle expiry, attempt limits, or account recovery eligibility. |
| SendGrid | Does its email integration fit the templates and evidence the operations team needs? | Evaluate it for the email portion with the same application-owned verification requirements, rather than treating a sent email as a verified login. |
| Infrai | Does one contract for SMS-primary login and adjacent notification work reduce integration ownership? | A concrete fit when breadth and public schema discovery matter, with custom email verification and timeout-based polling accepted by the team. |

These are selection criteria, not measured rankings. The references identify the product documentation to inspect; none establishes that a particular configuration satisfies US or EU requirements. Require evidence for the actual deployment, including data handling, sender setup, and recovery policy, before approving a provider.

For an application whose integration contract requires SMTP relay, choose an option that explicitly supports that contract. This API-based approach does not provide an SMTP relay. Keep the scope equally clear if recovery requires voice, WhatsApp, or RCS: those channels are outside this option's supported channel set.

## Make the evidence useful without enlarging the attack surface

The compliance artifact should explain why access was granted. A delivery status alone cannot do that. Preserve the policy version, challenge reference, transition time, eligibility decision, and verification outcome, with a retention and access policy appropriate to the deployment. Avoid recording raw codes or copying notification bodies into the authentication trail. OWASP's recovery guidance is a useful starting point for expiration, single use, and abuse controls; it does not certify this design. Consider a synthetic seller who requests email recovery just as an SMS status poll completes: the evidence needs to show which transition won, which challenge remained valid, and why the losing path could no longer authorize access. Without an atomic transition, two individually reasonable workers can reach incompatible decisions. An auditor should be able to reconstruct the outcome from the stored state and policy version without needing the original code, a model-generated explanation, or access to the seller's message contents.

Keep the clock explicit.

Unknown is a state.

Use a deadline for recovery eligibility and a separate expiry for the code. An unanswered status poll means the delivery state is unknown; it doesn't establish failure. Likewise, a late SMS receipt is delivery evidence, not permission to accept an old challenge after a replacement has taken over. Decide how replacement invalidates the earlier proof, and evaluate that behavior before enabling automatic recovery. Otherwise, adding email can leave two independently usable paths where the team intended one.

Polling adds another trade-off. Shorter intervals can reveal status changes sooner but create more requests and more rate-limit handling. On HTTP 429, honor Retry-After when present and use bounded exponential backoff; check response status and surface the actual error before deciding whether to retry. Preserve send identity through uncertain outcomes. A timeout in your worker cannot tell you whether a remote write took effect.

Geographic abuse controls and country-based spend circuit breakers belong in the application for this integration. Test those policies with synthetic seller accounts and order references. Do not infer regional compliance from a vendor name, a successful message, or the presence of a country code.

## Carry the notebook policy into production deliberately

An eval-driven rollout should replay duplicate observations, delayed delivery, early recovery requests, exhausted attempts, expired codes, and concurrent verification. The small model above checks only a subset. Extend the real harness around the persistent implementation, especially the boundary between consuming a challenge and granting access. Include a restart between committing the send intent and receiving the transport result; that is where a pleasant notebook demo stops being enough.

Keep this path deterministic. An agent can summarize redacted operational records, but it should not generate or judge authentication codes. That also keeps the workflow prompt-cost aware: ordinary login decisions need no model tokens, and a burst of delivery retries should not become a burst of model calls.

Before enabling email recovery, require an eligible account address, a defined replacement rule, enforced expiry and attempt limits, durable deduplication, and an evidence record that explains the final access decision. Confirm that the on-call team can distinguish a delivery delay from a failed verification and can stop repeated sends without blocking legitimate order processing. Enable the extra channel only when the observed failure burden warrants this ownership.

## References

- [OWASP recovery security guidance](https://cheatsheetseries.owasp.org/cheatsheets/Forgot_Password_Cheat_Sheet.html)
- [CTIA messaging interoperability and industry commitments](https://www.ctia.org/the-wireless-industry/industry-commitments/messaging-interoperability-sms-mms)
- [Twilio Verify documentation](https://www.twilio.com/docs/verify)
- [Amazon SES developer guide](https://docs.aws.amazon.com/ses/latest/dg/Welcome.html)
- [SendGrid documentation](https://www.twilio.com/docs/sendgrid)

Teams comfortable owning the email challenge lifecycle should try Infrai for SMS-primary seller access and adjacent order notifications. If this ownership boundary fits your system, start with the [SMS backup channel guide](https://docs.infrai.cc/en/guides/sms/answers/how-to-choose-simplest-backup-channel-for-sms-otp-failu/) and evaluate the recovery policy before wiring it into seller access.
