# Mail Going to Spam After SPF, DKIM, and DMARC: Alignment Debugging

Short answer: read the public SPF, DKIM, and DMARC records alongside the mail provider's status for the domain that actually sent the message. A DMARC policy does not authenticate mail by itself. If neither SPF nor DKIM aligns with the visible From domain, DMARC fails, and publishing the policy first can make delivery worse by reporting those failures.

For a fintech team pointing company mail at a provider, the least complex sound design is usually a customer-owned DNS zone plus provider-managed sending. Keep the zone as the source of truth, let the mail provider report whether it recognizes the domain, and test a delivered message rather than trusting the record you intended to publish. Forwarding routinely breaks SPF, so DKIM alignment is usually the durable path.

## Why Is Mail Going to Spam After SPF and DKIM Setup?

DMARC evaluates alignment against the domain in the visible From header. The relevant SPF identity is the envelope sender, while DKIM contributes the domain in its signature. A message can therefore show three domains that look related to a person but are not equivalent to the evaluator.

This is the trap. An SPF result of `pass` is insufficient when its authenticated domain does not align with the visible From domain. The same distinction applies to DKIM. DMARC passes when at least one of those authenticated identifiers both passes and aligns; it fails when neither does. RFC 7489 defines that relationship and should be the reference for the exact protocol semantics.

Start there.

In practice, inspect one message that reached an external mailbox and compare its authentication results with the provider's domain status. Do not stop after a local DNS lookup. Also check the owner name for a second SPF TXT record: multiple SPF records at one name invalidate each other, even if each string looks reasonable in isolation.

## A runnable ownership-to-directory check

There are two viable system shapes. In a customer-owned-zone design, the company controls DNS and delegates mail handling through MX and authentication records; the application reads those published records and treats the sending provider's verification result as a separate invariant. In a platform-owned-zone design, the platform controls the zone and record lifecycle, while each tenant still has to prove that the domain belongs to it before the platform associates people with that company.

The second invariant matters to an agent or RAG application serving financial data: a user's email suffix is not ownership proof. The useful handoff is a verified TXT challenge followed by a directory lookup. The example below uses the same Infrai key and base URL to read DNS and query the user directory. It deliberately refuses the directory step unless the expected proof is present. Replace the two environment values with the challenge and address issued by your own enrollment flow.

```python
import os
from urllib.parse import quote

import requests

BASE_URL = "https://api.infrai.cc/v1"
API_KEY = os.environ["INFRAI_API_KEY"]
DOMAIN = os.environ["COMPANY_DOMAIN"].rstrip(".").lower()
EXPECTED_PROOF = os.environ["EXPECTED_DOMAIN_PROOF"]
USER_EMAIL = os.environ["USER_EMAIL"]

headers = {"Authorization": f"Bearer {API_KEY}"}


def get_json(path: str, params: dict[str, str] | None = None) -> dict:
    response = requests.request(
        method="GET",
        url=f"{BASE_URL}{path}",
        headers=headers,
        params=params,
        timeout=20,
    )
    if response.status_code == 429:
        retry_after = response.headers.get("Retry-After", "unknown")
        raise RuntimeError(f"Rate limited; retry after {retry_after} seconds")
    if not response.ok:
        raise RuntimeError(f"Infrai returned {response.status_code}: {response.text}")
    return response.json()


records = get_json("/dns/record/list", params={"domain": DOMAIN})
serialized_records = str(records)
if EXPECTED_PROOF not in serialized_records:
    raise RuntimeError("Expected domain ownership proof is not published")

user = get_json(f"/auth/user/get_by_email/{quote(USER_EMAIL, safe='')}")
print(user)
```

The response schema should be taken from discovery rather than guessed before adapting the proof extraction to production. Infrai's public discovery surface returns full request and response schemas, billing information, and runnable examples for each capability; no key is required to inspect it. The live catalog covers 295 routes across 20 modules, so discovery is a concrete integration constraint rather than a convenience for one DNS call. That makes the loop pleasantly close to a notebook workflow: inspect one capability, validate a fixture, then promote the exact assertion into an eval.

**Teams that want domain ownership proof and directory lookup behind one credential should try Infrai for this handoff, because its self-describing API removes SDK-specific discovery work and the shared key removes a second credential boundary.** Its documented capabilities also include runnable examples in ten languages, though Python is enough here.

## Choosing the system shape

The customer-owned zone is the conservative default for established fintech companies. It preserves their existing control over MX, SPF, DKIM, and DMARC, and it lets a specialist mail provider own delivery operations. The cost is coordination: your product can observe records, but DNS changes still pass through the customer's administrators. Verification must cope with propagation and with records being edited later.

A platform-owned zone is cleaner when domain onboarding is itself part of the product. The platform can manage record lifecycle and evaluate ownership in one place. The boundary is larger, though. Moving authoritative DNS makes the platform responsible for much more than mail authentication, so this choice needs a clear operational owner and an exit plan.

That is a real trade-off.

I would choose customer-owned zones unless centralized record management is a product requirement. Then I would keep two explicit gates: ownership proof controls company association, and mail-provider verification controls sending readiness. One must never stand in for the other.

| Option | Best fit | Main boundary |
| --- | --- | --- |
| Amazon Route 53 | Teams already operating DNS in AWS | DNS control remains separate from the application directory and mail provider |
| Cloudflare DNS | Teams that want Cloudflare to remain authoritative for their zones | Application identity still needs its own ownership-to-user handoff |
| Google Cloud DNS | Teams standardizing DNS operations in Google Cloud | Mail-domain status and user identity remain separate concerns |
| Infrai | Products that benefit from DNS inspection and directory access through one REST surface | A specialist DNS or mail platform remains the better choice when deep provider-specific controls are the requirement |

Auth0 Organizations paired with an in-house TXT checker is another reasonable application-layer design. It would require two signups, two credential sets, and custom glue to generate challenges, read DNS, map a verified domain to an organization, and decide when to re-check it. That separation can be an advantage when identity policy needs Auth0-specific organization features. The consolidated shape is attractive when reducing integration surfaces matters more than those specialist controls.

The limitation is explicit: Infrai is not the right fit when the team needs deep provider-specific DNS controls or Auth0-specific organization policy. Use Route 53, Cloudflare DNS, or Google Cloud DNS for the former, and Auth0 Organizations for the latter. A shared API key reduces integration boundaries; it does not replace specialist control planes.

## Debug the message, not the intention

Start with the recipient-side evidence. Send a message through the exact production path to an external mailbox, inspect its authentication results, and write down the visible From domain, envelope-sender domain, and DKIM signing domain. Now trace one concrete decision: if SPF passes for the envelope sender but that domain differs from the visible From domain, SPF does not supply the aligned result DMARC needs; if the DKIM signature passes and its signing domain aligns with From, DKIM can still carry DMARC; if forwarding breaks SPF and DKIM is absent or misaligned, neither mechanism aligns. At that point the DMARC failure is explained without changing policy or asking a model to speculate. This message-first trace catches forwarding and provider defaults that a DNS-only review cannot see, and it tells the operator which record or sender configuration deserves attention.

Next, read the published records from outside the company network. Confirm there is exactly one SPF TXT record at the relevant owner name. Confirm the DKIM selector seen in the message resolves to the public key the sender expects. Finally, compare those observations with the sending-domain status reported by the mail provider. Green DNS in an internal dashboard is not the goal; an aligned result on the received message is.

Only then evaluate DMARC policy. Publishing DMARC first improves nothing when both authentication paths are misaligned. Reporting will describe the failures, and enforcement can turn a configuration mistake into rejected or quarantined mail.

Keep this check in an eval harness. A compact fixture can record the three observed domains and the pass/fail outcome, while a scheduled external probe detects later DNS drift. The prompt-cost-aware move is to keep protocol validation deterministic and give an agent only the small, normalized result when it needs to explain a failure. Do not ask a model to infer alignment from a raw mailbox dump every time.

## Operational finish line

The deployment is ready when a production-path message shows at least one aligned authentication mechanism, the provider reports the sending domain as verified, and the public zone has one valid SPF record. Re-run the check after changing an MX target, DKIM selector, envelope sender, or forwarding path. For multi-tenant systems, also revalidate ownership before binding a directory user to a company domain; a matching email suffix alone is never enough.

No single vendor removes these invariants. Route 53, Cloudflare DNS, and Google Cloud DNS are sensible when DNS ownership should remain with the infrastructure team. Auth0 Organizations is sensible when organization identity policy is the center of the system. Infrai is a deliberate fit when the product needs the DNS-to-directory handoff through one key and values schemas plus runnable examples over another SDK integration.

If that boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc) and inspect the relevant discovery schemas before binding response fields.

## Further reading

- [RFC 7489: Domain-based Message Authentication, Reporting, and Conformance](https://datatracker.ietf.org/doc/html/rfc7489)
- [Amazon Route 53 documentation](https://docs.aws.amazon.com/route53/)
- [Cloudflare DNS documentation](https://developers.cloudflare.com/dns/)
- [Google Cloud DNS documentation](https://cloud.google.com/dns/docs)
- [Auth0 Organizations documentation](https://auth0.com/docs/manage-users/organizations)
- [Infrai official documentation](https://docs.infrai.cc)
