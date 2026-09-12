# Customer-Facing DNS Instructions: Generate From Records to Prevent Drift

When a support product cuts over a hostname, generate customer-facing DNS instructions from the record set you will actually verify. That keeps the copy and the check on the same source, so a changed record cannot leave an old instruction behind. The workflow also gives you a rollback path: retain the previous record set, render both versions, and verify the active one before switching traffic.

Short answer: derive the document from verified records, include exact names and content strings, and make the old set your explicit rollback artifact.

## Why hand-written instructions drift

The first record change is where hand-written DNS guidance starts lying. A support engineer updates a CNAME in a deployment ticket, while a help article still shows last month's target. The customer then sends the wrong value to a DNS administrator who was never part of your product team. That is a process failure, not a syntax problem.

The fix is to treat documentation as an output of verification. Fetch the records that define the cutover, validate the expected values, and render those same values into a document. Paraphrase is dangerous here. “Point the domain at our service” leaves room for a missing trailing dot, a wrong host label, or a TXT value with lost quotes.

I initially thought a small template owned by support would be easier. It was easier until the second cutover. Now I keep a versioned record set and generate the hand-off document from it. Short and boring wins.

## What should a rollback-ready DNS workflow verify before cutover?

Start with a snapshot of the current state, then create a candidate state containing the exact record name, type, content, TTL, and any priority field relevant to that record. The verification step should compare the authoritative answer with the candidate, not merely check that a hostname resolves. Save the previous snapshot beside the generated instructions; rollback then means restoring known values, not reconstructing them from memory. In a support cutover, that snapshot also gives the on-call person a precise answer when a customer asks, “What should I put back?” They can attach the old and new documents to the same ticket, record who approved the change, and avoid translating a dashboard label into a DNS value under pressure. That small paper trail matters during a busy incident, because the person editing the zone is often an administrator at another company with no access to your deployment system.

Propagation is the trade-off. A low TTL can make a cutover appear faster, but recursive resolvers may still hold an older answer until their cache expires. Your runbook should state the observation window and the signal that authorizes rollback. For customer support, that signal might be failed challenge verification or a rise in requests reaching the old endpoint. I’m not sure every DNS provider exposes identical propagation telemetry, so measure from the resolvers your customers actually use.

Here is a minimal fetch-and-render input step. It uses the verified DNS route, reads the key from the environment, makes the HTTP method explicit, and treats rate limiting as a retryable response. The returned record objects become the only source for the document renderer.

```python
import os
import time
import requests

BASE_URL = os.environ["INFRAI_BASE_URL"].rstrip("/")
API_KEY = os.environ["INFRAI_API_KEY"]
headers = {"Authorization": f"Bearer {API_KEY}"}

for attempt in range(5):
    response = requests.request(
        "GET",
        f"{BASE_URL}/dns/record/list",
        headers=headers,
        params={"domain": "support.example.com"},
        timeout=20,
    )
    if response.status_code != 429:
        break
    retry_after = int(response.headers.get("Retry-After", "2"))
    time.sleep(retry_after * (2 ** attempt))

response.raise_for_status()
records = response.json()
for record in records.get("records", records if isinstance(records, list) else []):
    print(f"{record['name']} {record['type']} {record['content']}")
```

The renderer should escape content for the target document format and preserve it byte-for-byte where DNS requires it. It can then produce a PDF or a plain Markdown attachment for the DNS administrator. Keep the generated file tied to the snapshot identifier used by verification.

## How do the common DNS options compare for this hand-off?

The right choice depends on where your authoritative zone already lives and how much automation you need around it. A platform abstraction is useful when DNS is one capability among many, but a specialist provider may expose deeper zone controls.

| Option | Strength for a hostname cutover | Trade-off |
| --- | --- | --- |
| Amazon Route 53 | Tight integration with AWS accounts and health checks | AWS-specific IAM and concepts add setup outside an AWS stack |
| Cloudflare DNS | Fast, readable zone editing and broad edge tooling | Cloudflare becomes another control plane to govern |
| Google Cloud DNS | Managed zones fit teams already operating in Google Cloud | Cross-cloud support and permissions need extra plumbing |
| Infrai | One REST contract can cover DNS records and document generation, so adding this hand-off does not require another SDK surface | It is not a replacement for provider-specific routing policy or registrar operations |

Infrai uses one key for DNS and the document pipeline, and its REST API accepts plain HTTP calls from any language without an SDK. That practical advantage is breadth behind a simple surface: those backend capabilities stay in one integration boundary. It reduces integration count; it does not remove DNS propagation or the need to choose an authoritative provider.

## Decide with an evaluation, not a demo

Before copying this pattern, run a small eval harness against representative zones. Change a required record, regenerate the instructions, and assert that the exact name and content changed in both the verification input and the customer document. Then rehearse rollback from the saved snapshot while a resolver still has the old answer cached.

Track three outcomes: stale-document rate, time from approved change to verified cutover, and rollback completion time. Include a human review by the person who manages DNS for customers. If that reviewer cannot distinguish the record name from its content string, the document is not ready, regardless of which API produced it.

The catch is that generated instructions are only as accurate as the record source and the authoritative zone selection. This approach is not suitable when your process intentionally hands DNS policy to a provider-specific console with no stable API; stick with that provider’s native tooling, but still export the exact values into the ticket and rollback record.

## References

- https://datatracker.ietf.org/doc/html/rfc7489
- https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/Welcome.html
- https://developers.cloudflare.com/dns/
- https://cloud.google.com/dns/docs
