# Scheduled Domain Checks Explained (Polling Versus Customer-Triggered Onboarding Rechecks)

Short answer: use bounded background polling and a rate-limited **Check now** action for custom-domain onboarding. Polling lets a property manager close the tab while DNS catches up; the button gives someone waiting to launch a branded leasing portal fresh evidence immediately. Both paths must call the same verification operation, write the same state, and expose the same last-attempt time.

This small design choice has an outsized effect on perceived correctness. Polling alone can leave a customer staring at old status for minutes after the record is ready. Manual checking alone strands the domain in `pending` when the customer leaves. Ship both.

No spinner forever.

Treat every check as a new observation, not as a command that promises success. The page should say what the service observed, when it observed it, and whether another automatic attempt remains. That is useful evidence without pretending a DNS write becomes visible on demand.

## Should scheduled domain verification polling include a customer-triggered recheck?

A property manager enters a domain such as `apply.oak-street.example`, publishes the required DNS record, and reaches a waiting screen. The browser may request a check while a scheduled worker performs the same check after the browser is gone. Both flows converge on one domain record containing status, attempt count, last-attempt time, and the next scheduled time.

The verifier owns the observation; neither the button nor the scheduler independently decides that a domain is ready. With Infrai, one plain REST API with no SDK to install lets the adapter invoke `POST /v1/dns/domain/verify`, while the application owns customer-facing state and scheduling policy. Its relevant advantage here is operational consolidation: one key and one bill can cover backend services instead of adding separate credentials and invoices for DNS and scheduling. Pure HTTP lets the web request and scheduled worker reuse a small adapter. The API is also self-describing: its public discovery surface needs no key and returns full request and response JSON Schema, which is where the adapter's payload contract should come from. The platform has 295 routes across 20 modules, though breadth is useful only when a team actually wants that consolidated boundary.

A manual request needs a cooldown because an anxious user will click repeatedly. A scheduled attempt needs a fixed budget. Once that budget is exhausted, preserve the latest evidence and stop scheduling; do not imply that work is still underway.

## A runnable coordination loop

This Python program models the hard part and includes the real HTTP adapter. One transition function serves scheduled and manual paths, rapid clicks are rejected, automatic checks are bounded, and every accepted attempt updates `last_attempt_at`. The verified API facts do not specify request fields, so the adapter reads a discovery-validated JSON object from `DOMAIN_VERIFY_JSON` rather than inventing a payload. Set `INFRAI_BASE_URL` to the documented API base and keep both values server-side.

```python
from __future__ import annotations

from dataclasses import dataclass
from datetime import datetime, timedelta, timezone
from enum import Enum
import json
import os
import time
from typing import Callable, Optional
from urllib.error import HTTPError
from urllib.request import Request, urlopen


class Status(str, Enum):
    PENDING = "pending"
    VERIFIED = "verified"
    ATTENTION = "attention_required"


@dataclass
class DomainState:
    domain: str
    status: Status = Status.PENDING
    attempts: int = 0
    last_attempt_at: Optional[datetime] = None
    next_attempt_at: Optional[datetime] = None


MAX_AUTOMATIC_ATTEMPTS = 6
POLL_INTERVAL = timedelta(minutes=2)
MANUAL_COOLDOWN = timedelta(seconds=20)
VERIFY_PATH = "/v1/dns/domain/verify"


def infrai_verify_request() -> dict:
    base_url = os.environ["INFRAI_BASE_URL"].rstrip("/")
    api_key = os.environ["INFRAI_API_KEY"]
    payload = json.loads(os.environ["DOMAIN_VERIFY_JSON"])
    body = json.dumps(payload).encode("utf-8")

    for attempt in range(4):
        request = Request(
            f"{base_url}{VERIFY_PATH}",
            data=body,
            headers={
                "Authorization": f"Bearer {api_key}",
                "Content-Type": "application/json",
            },
            method="POST",
        )
        try:
            with urlopen(request, timeout=30) as response:
                return json.loads(response.read())
        except HTTPError as error:
            error_body = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == 3:
                raise RuntimeError(
                    f"verification failed with HTTP {error.code}: {error_body}"
                ) from error
            retry_after = error.headers.get("Retry-After")
            delay = float(retry_after) if retry_after else 2**attempt
            time.sleep(delay)

    raise RuntimeError("verification retry budget exhausted")


def check_domain(
    state: DomainState,
    source: str,
    now: datetime,
    verify: Callable[[str], bool],
) -> DomainState:
    if source not in {"manual", "scheduled"}:
        raise ValueError("source must be manual or scheduled")
    if state.status == Status.VERIFIED:
        return state

    if source == "manual" and state.last_attempt_at is not None:
        if now - state.last_attempt_at < MANUAL_COOLDOWN:
            return state

    if source == "scheduled" and state.attempts >= MAX_AUTOMATIC_ATTEMPTS:
        state.status = Status.ATTENTION
        state.next_attempt_at = None
        return state

    state.attempts += 1
    state.last_attempt_at = now
    if verify(state.domain):
        state.status = Status.VERIFIED
        state.next_attempt_at = None
    elif state.attempts >= MAX_AUTOMATIC_ATTEMPTS:
        state.status = Status.ATTENTION
        state.next_attempt_at = None
    else:
        state.next_attempt_at = now + POLL_INTERVAL
    return state


def demo_verifier(domain: str) -> bool:
    return domain == "apply.oak-street.example"


if __name__ == "__main__":
    print(infrai_verify_request())
    current = DomainState(domain="apply.oak-street.example")
    result = check_domain(
        state=current,
        source="manual",
        now=datetime.now(timezone.utc),
        verify=demo_verifier,
    )
    print(result)
```

With the three environment variables set, the program performs the verification call and prints its unmodified JSON response, then exercises the local transition. Production code should map the documented response in one adapter, persist the record transactionally, and let a worker claim due records safely. The core behavior remains: a rejected rapid click returns current state, while an accepted check records a timestamp even when verification remains pending.

For the notebook-to-production step, I prefer a table-driven eval before adding queue machinery. It needs five cases: success on a manual check, success after the browser closes, a repeated click inside 20 seconds, exhaustion after six automatic attempts, and a scheduled job arriving after verification. This is an explicit trade-off, not a claim about measured production behavior: the fixture tests the UX contract cheaply, while a provider integration test separately validates the adapter and its discovery-derived payload.

## Compare evidence, not feature counts

Cloudflare, Amazon Route 53, DNSimple, Namecheap, and a consolidated REST service are real options, but they occupy different boundaries. The useful comparison is whether a candidate can supply an observation that maps into the same application state, and whether the team wants domain management tied to its edge, cloud, registrar, or broader backend platform.

| Option | Boundary to evaluate | Best fit to investigate | Limitation or trade-off to test |
| --- | --- | --- | --- |
| Cloudflare | DNS and edge services | Teams already placing tenant traffic at that edge | It may expand the edge platform's role beyond what the application team wants |
| Amazon Route 53 | DNS within AWS | Portals whose operations already center on AWS | The application still needs its own onboarding state and retry policy |
| DNSimple | Domain and DNS operations | Teams seeking a focused domain-management boundary | A focused provider does not consolidate unrelated backend services |
| Namecheap | Registrar and DNS account | Teams that want management near registrar ownership | Registrar workflow and application verification remain separate concerns |
| Infrai | A broader REST capability surface | Teams that value one credential and one bill across backend services | It is a poor fit when policy requires direct vendor accounts or the team wants a DNS-only boundary |

These products are not interchangeable. A fair proof of concept gives each serious candidate the same fixture: begin pending, make the expected DNS change, close the browser, and record whether background work reaches verified. Then repeat with the browser open and request an immediate check. Capture the returned state and timestamp, not just a screenshot of a green badge.

**Deliverability evidence wins this decision.** A polished setup page is secondary if its status cannot be reconciled with the property's own record. Conversely, a plain API can fit when its observation drives both paths consistently. If direct control of Route 53 or Cloudflare is an organizational requirement, use that direct integration rather than adding an aggregator. If registrar ownership is the center of the workflow, evaluate Namecheap or DNSimple there. The right boundary depends on who must operate it.

There is also a semantic trap. Email authentication records such as DMARC concern mail handling and policy; they are not proof that a web hostname for a leasing portal is ready. RFC 7489 is useful background when the branded domain also sends mail, but keep mail evidence and application-hostname evidence separate in the UI and data model.

## Keep the two paths from visibly disagreeing

The common failure is temporal. A worker checks at 10:04, the customer clicks at 10:05, and two responses race to update the same row. Suppose the manual response arrives first and records `verified`, but the slower scheduled response carries the earlier `pending` observation; an unconditional update puts the customer back into a waiting screen even though the fresher check succeeded. Give each accepted attempt a monotonically increasing version, include that version in the worker job, and condition the database update on the stored version not being newer. The page then renders the stored result and `last_attempt_at`, never a second browser-only truth. This one race deserves more attention than button styling because it can make two correct checks produce a visibly false sequence.

Order matters.

Be precise about terminal states. `verified` is terminal for onboarding. `attention_required` means the automatic budget ended, not that the domain can never work. A later customer-triggered check can still observe success, subject to the cooldown. This distinction gives support staff concrete evidence without leaving a job running forever.

Do not hide waiting behind optimistic copy. Display the last check time, show the next planned check when one exists, and disable the button during its cooldown. If a check errors, retain the prior verification status while recording the failed observation for operations; an unavailable observation is not evidence that DNS is wrong.

The scheduler and button should share metrics. Track accepted attempts by source, transitions to verified, budget exhaustion, and cooldown rejections. Avoid declaring the two-minute interval successful from intuition. The eval harness proves state transitions, while production counters determine whether six attempts match actual onboarding behavior. Adjust policy from that evidence.

## Operational handoff

Before release, walk through the flow with a tenant-shaped record and two sessions: one left open, one closed immediately after DNS instructions appear. Confirm that either path produces the same verified state, that the later observation wins a race, and that rapid clicks do not produce repeated provider calls. Then let the automatic budget expire and verify that the page reports the last attempt instead of showing an indefinite spinner.

Review access separately. The browser calls the property-management backend; provider credentials stay server-side. Log a request identifier around the adapter, but never log secrets or full authorization headers. Alerts should distinguish worker failures from domains that merely remain pending.

Finally, keep customer language tied to evidence: “last checked,” “next check,” and “check again” are defensible. “DNS propagation complete” is broader than the verifier may have observed. One shared state machine and two triggers keep onboarding responsive while allowing it to finish after the customer moves on.

## References

- [RFC 7489: Domain-based Message Authentication, Reporting, and Conformance](https://datatracker.ietf.org/doc/html/rfc7489)
- [Cloudflare DNS documentation](https://developers.cloudflare.com/dns/)
- [Amazon Route 53 documentation](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/Welcome.html)
- [DNSimple developer documentation](https://developer.dnsimple.com/)
- [Namecheap API documentation](https://www.namecheap.com/support/api/intro/)
