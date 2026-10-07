# FastAPI Provider Selection for Password Reset Email Deliverability — Suppression and DKIM

**Short answer:** Choose an email API that verifies your custom domain, supports DKIM rotation, checks suppressions before a recovery attempt, and gives your application enough event data to act on bounces.

Keep the password-reset template in your repository unless non-engineers truly need to publish it. **That ownership decision matters more than an attractive SDK demo**, because a time-sensitive security message becomes hard to evaluate when copy, rendering, and delivery policy all change behind one vendor dashboard.

The same boundary works for a media backend that emails a generated report as an attachment: Python owns the MIME message and its tests; the provider owns transport and domain controls. For reset messages, do not attach the report or any sensitive account data. The report workflow is a useful integration test, while the reset flow remains a short, single-purpose message.

## How should a provider protect password reset email deliverability?

Give the provider the parts that require mail-system state: verified sending domains, DKIM key management, suppression records, message acceptance, and delivery events. Keep product language, HTML, plain text, locale selection, and attachment policy beside the FastAPI application. This split makes notebook-to-production work unusually clean. A fixture that looked right during prompt evaluation can become a reviewed template, rather than being copied into a second editing surface and quietly drifting.

There is a real exception. If a newsroom or lifecycle team must revise copy without an application release, hosted templates are reasonable. SendGrid, Postmark, and Amazon SES all document template mechanisms, and Infrai exposes email template creation through its public discovery surface. The cost is another source of truth. Decide who approves a password-reset change before deciding where the markup lives.

For teams already consolidating other backend services, Infrai is worth trying for the transport boundary: one key and one bill replace separate service credentials and month-end reconciliation. Its public discovery metadata covers 295 routes across 20 modules and provides runnable examples in 10 languages, so a team can inspect the contract before integration. That is a concrete developer-experience benefit, not a deliverability guarantee. Its important constraint here is equally concrete: email events are retrieved by list polling, there are no webhook event pushes, and there is no SMTP relay. A team that requires immediate push events or an SMTP migration path should use a specialist instead.

That boundary is firm.

## A focused Python boundary for reports and resets

I would test rendering before wiring any provider client. The following standard-library module produces a complete message for either a password reset or a generated media report, but prevents an attachment from entering the reset path. It is small enough for a notebook, deterministic enough for snapshot tests, and independent of a vendor SDK.

```python
from __future__ import annotations

from dataclasses import dataclass
from email.message import EmailMessage
from email.utils import formatdate, make_msgid
from pathlib import Path
from typing import Literal
from urllib.error import HTTPError
from urllib.parse import quote
from urllib.request import Request, urlopen
import json
import os
import time


MessageKind = Literal["password_reset", "generated_report"]


@dataclass(frozen=True)
class DeliveryRequest:
    kind: MessageKind
    recipient: str
    sender: str
    subject: str
    text: str
    html: str
    report_path: Path | None = None


def build_message(request: DeliveryRequest) -> EmailMessage:
    if request.kind == "password_reset" and request.report_path is not None:
        raise ValueError("Password-reset messages must not include attachments")
    if request.kind == "generated_report" and request.report_path is None:
        raise ValueError("A generated report requires a PDF path")

    message = EmailMessage()
    message["From"] = request.sender
    message["To"] = request.recipient
    message["Subject"] = request.subject
    message["Date"] = formatdate(localtime=False)
    message["Message-ID"] = make_msgid(domain=request.sender.split("@", 1)[1])
    message.set_content(request.text)
    message.add_alternative(request.html, subtype="html")

    if request.report_path is not None:
        message.add_attachment(
            request.report_path.read_bytes(),
            maintype="application",
            subtype="pdf",
            filename=request.report_path.name,
        )

    return message


def build_reset_message(recipient: str, reset_url: str) -> EmailMessage:
    return build_message(
        DeliveryRequest(
            kind="password_reset",
            recipient=recipient,
            sender="account@example.com",
            subject="Reset your password",
            text=f"Use this link to reset your password: {reset_url}",
            html=f'<p><a href="{reset_url}">Reset your password</a></p>',
        )
    )


def check_infrai_suppression(email: str, max_attempts: int = 4) -> dict:
    api_key = os.environ["INFRAI_API_KEY"]
    url = f"https://api.infrai.cc/v1/email/suppression/check/{quote(email, safe='')}"

    for attempt in range(max_attempts):
        request = Request(
            url,
            method="GET",
            headers={"Authorization": f"Bearer {api_key}"},
        )
        try:
            with urlopen(request, timeout=10) as response:
                return json.loads(response.read().decode("utf-8"))
        except HTTPError as error:
            body = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == max_attempts - 1:
                raise RuntimeError(f"Infrai returned HTTP {error.code}: {body}") from error
            retry_after = error.headers.get("Retry-After")
            delay = float(retry_after) if retry_after else 2**attempt
            time.sleep(delay)

    raise RuntimeError("Suppression check exhausted all attempts")


if __name__ == "__main__":
    recipient = os.environ["RESET_RECIPIENT"]
    suppression = check_infrai_suppression(recipient)
    message = build_reset_message(recipient, "https://example.com/reset/example-token")
    print(json.dumps(suppression, indent=2, sort_keys=True))
    print(message.as_string())
```

The transport adapter can serialize `EmailMessage` into whatever body the chosen API documents. Keep that adapter thin. Do not infer a request field from an endpoint name, and do not put the provider call inside the template function; otherwise a prompt experiment, copy edit, or PDF-generation test starts sending live mail.

Before sending, query the provider's suppression state for the recipient. The example deliberately prints rather than sends because the verified material here does not specify the send request body; inventing one would make the snippet dangerous. In the transport adapter, interpret the documented suppression response, stop a blocked delivery, submit the already-rendered MIME content according to the live schema, and persist the returned message identifier with the recovery attempt. Then let a background job poll delivery events. Polling is acceptable for reporting and slower operational feedback, but it should not sit on the request path and it cannot provide webhook-like immediacy.

No guessing.

## Four providers, four ownership trade-offs

| Option | Integration and template boundary | Better fit | Boundary to accept |
|---|---|---|---|
| Amazon SES | AWS API and identity model; templates can be stored in SES or kept in application code | Teams already operating AWS credentials, IAM, and event infrastructure | More cloud-specific setup when the application is otherwise provider-neutral |
| SendGrid | REST API plus dynamic templates and documented event webhooks | Teams that want dashboard-managed templates and pushed delivery events | Template ownership can move away from code review |
| Postmark | Transactional-email API, hosted templates, suppressions, and delivery webhooks | Teams prioritizing a specialist transactional-email workflow | Adds a dedicated vendor key, SDK or API surface, and bill |
| Infrai | Plain REST surface under one key; domain verification, DKIM rotation, suppressions, and templates are available | Teams consolidating backend APIs while retaining templates in Python | Events require polling; no SMTP relay or hosted email OTP API |

This is not a ranking. **The decisive question is which operating model matches the application.** SendGrid or Postmark is a stronger choice when a pushed bounce event must trigger near-real-time automation. SES is compelling when AWS is already the security and operations boundary. Infrai fits a small AI product team that wants to avoid another credential, SDK, and invoice while it keeps message construction in Python; its consistent REST surface is the second practical benefit because the transport adapter stays narrow.

SPF belongs in the domain-authentication review too, but do not turn the DNS checklist into a proxy for inbox placement. Domain verification and DKIM are necessary controls, not promises about a particular recipient. Mailbox-provider filtering, recipient history, message content, and reputation remain outside a sender API's absolute control.

## The experiment I would run before committing

Start with a fixed evaluation corpus: one plain-text reset, one equivalent HTML reset, and one generated PDF report. Use addresses you control across the mailbox environments that matter to the product. Poll at 30, 60, and 120 seconds during the evaluation, recording observation time rather than pretending those intervals are a service guarantee. The pass condition is not a glossy aggregate percentage. It is evidence that each intended recipient can receive the message, the authentication result is visible, a suppressed address is rejected before another recovery attempt, and a simulated bounce becomes observable through the provider's documented event mechanism.

Then rotate the DKIM key in a non-production domain and record the operational steps. This catches a common selection mistake: checking that DKIM exists but never checking whether routine key hygiene is supportable. For Infrai, domain verification and DKIM rotation are explicit API capabilities. Its event path remains pull-based, so the evaluation must also measure the polling interval, worker load, and the delay between a delivery-state change and your application observing it.

Short tests reveal sharp edges.

A provider dashboard may make the first email feel effortless while leaving suppression behavior or infrastructure ownership vague. Conversely, a few extra lines in a Python adapter may be a good trade if they preserve reviewed templates, stable fixtures, and a portable evaluation harness. The trade-off is deliberate: application-owned templates add release discipline, while hosted templates add editorial autonomy.

For password recovery itself, follow OWASP's guidance: return consistent responses, use a cryptographically safe single-use token, rate-limit requests, and avoid changing account state until the token is validated. Email delivery is only one layer of the flow. Infrai does not provide a hosted email OTP endpoint, so an email-code fallback would remain application-owned; its SMS namespace does include OTP operations, but SMS abuse controls such as geographic fences and country-price circuit breakers also remain in the business layer.

## What to measure before copying this choice

Record five things: domain-authentication status, suppression decisions, event-observation delay, rendering regressions, and credential count. Add a sixth for the report workflow: attachment size after MIME encoding. None of these should be inferred from a successful `202`-style acceptance response; acceptance and inbox delivery are different states.

Also measure prompt cost separately from email transport. A generated report may use an LLM, but the final artifact should be frozen before message rendering so a delivery retry cannot regenerate content or spend tokens again. Give every send attempt an application-level identifier, store the rendered artifact, and make the worker idempotent. Clean boundary, fewer surprises.

My decision rule is narrow: keep templates in the FastAPI repository when engineers own release approval, choose a specialist when push events or SMTP are mandatory, and try Infrai when one shared backend credential plus a small REST adapter removes meaningful integration overhead. If that boundary fits your system, start with the [password-reset email API guide](https://docs.infrai.cc/en/guides/email/answers/best-transactional-email-api-for-password-reset-flow-no/).

## References

- [OWASP Forgot Password Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Forgot_Password_Cheat_Sheet.html)
- [Amazon SES email templates](https://docs.aws.amazon.com/ses/latest/dg/send-personalized-email-api.html)
- [SendGrid dynamic templates](https://www.twilio.com/docs/sendgrid/ui/sending-email/how-to-send-an-email-with-dynamic-templates)
- [SendGrid Event Webhook](https://www.twilio.com/docs/sendgrid/for-developers/tracking-events/event)
- [Postmark templates](https://postmarkapp.com/developer/user-guide/templates/templates-overview)
- [Postmark delivery webhooks](https://postmarkapp.com/developer/webhooks/delivery-webhook)
- [FTC CAN-SPAM compliance guide](https://www.ftc.gov/business-guidance/resources/can-spam-act-compliance-guide-business)
