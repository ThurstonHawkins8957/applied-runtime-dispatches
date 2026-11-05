# Python DNS Drift Checks — Custom Domain Onboarding for Property SaaS Customers

Get each property manager's sending domain into an explicit desired-state manifest, then compare that manifest with DNS observations before enabling mail. TL;DR: use per-tenant verification and record checks for customer domains; treat wildcard routing as a separate application concern. For rent reminders and maintenance updates, the useful onboarding decision is whether published authentication still matches the tenant's approved sending configuration.

Authentication isn't delivery assurance. RFC 7489 defines how DMARC connects the visible From domain to authenticated identifiers and publishes handling policy; it does not promise inbox placement. A green DNS comparison is narrower still: it says that a particular observation matched your intended records. Keep those boundaries in the status you show a property manager.

The data flow can stay small: store approved tenant intent, collect DNS answers with their observation context, compare the two deterministically, and attach the result to that intent revision. A separate mail check evaluates authentication and alignment on an actual message. This gives a notebook-to-prod workflow a clear boundary: Python can explain a mismatch without asking a language model to decide whether a domain is ready.

## Start with a runnable drift check

This example uses synthetic property-management data and Python's standard library. It runs offline.

The dictionaries represent an approved manifest and a normalized DNS snapshot, not a resolver implementation or a live customer deployment. The SPF and DKIM strings are deliberately illustrative; the public-key field contains a placeholder, and none of these records should be published as a working mail configuration.

```python
from dataclasses import dataclass


@dataclass(frozen=True)
class Record:
    owner: str
    kind: str
    value: str


domain = "cedar-properties.example"
revision = 7
desired = (
    Record(domain, "spf", "v=spf1 -all"),
    Record(
        f"leasing._domainkey.{domain}",
        "dkim",
        "v=DKIM1; k=rsa; p=PLACEHOLDER",
    ),
    Record(
        f"_dmarc.{domain}",
        "dmarc",
        "v=DMARC1; p=none",
    ),
)

# Each tuple contains complete TXT record values, after chunk assembly.
# An empty tuple means an observed absence, never a lookup timeout.
observed = {
    domain: ("v=spf1 -all",),
    f"leasing._domainkey.{domain}": (
        "v=DKIM1; k=rsa; p=PLACEHOLDER",
    ),
    f"_dmarc.{domain}": ("v=DMARC1; p=reject",),
}


def compare(records, snapshot):
    results = []
    for record in records:
        values = snapshot.get(record.owner)
        if values is None:
            status = "unobserved"
        elif not values:
            status = "missing"
        elif values == (record.value,):
            status = "matches_intent"
        else:
            status = "review_required"
        results.append((record.kind, record.owner, status))
    return results


results = compare(desired, observed)
print(f"Intent revision: {revision}")
for kind, owner, status in results:
    print(f"{kind}: {owner}: {status}")
assert [row[2] for row in results] == [
    "matches_intent",
    "matches_intent",
    "review_required",
]
```

The last result matters. The snapshot contains a DMARC record, but its policy differs from the approved intent. Presence alone would hide that change. Here, revision 7 expects monitoring policy and observes a rejection policy; review is appropriate even though the published policy sounds stricter.

Compare the intent.

Exact equality is a deliberately conservative starting point. This example sends unrelated TXT values, alternate serialization, and multiple answers to review. That can produce extra work. A production parser can reduce those false alarms, but it needs protocol-specific validation before it declares two representations equivalent. Removing spaces indiscriminately or choosing the first TXT answer would make a short script look more capable while erasing useful evidence.

The trade-off is review effort: exact matching is unsuitable for automatically accepting every semantically equivalent record. Its limitation is intentional. Choose a protocol-aware validator when automatic acceptance matters more than keeping the comparison small and inspectable.

## How should SaaS customers verify DNS records during custom domain onboarding?

Use separate acceptance criteria for application routing, domain control, and mail authentication. A wildcard routing configuration is not evidence that the tenant has approved the particular SPF, DKIM, and DMARC records your sending configuration requires. Even if a hostname reaches the application, that observation does not evaluate a message's authenticated identifiers or DMARC alignment.

For this property-management system, choose per-tenant intent because the decision concerns a specific customer domain and its approved mail configuration. Store the sending domain, the intended DKIM selector, the expected records, and the approval revision together. Bind domain-control evidence to that same tenant and intended domain. Routing can have its own wildcard strategy; it should not supply the mail-ready flag.

There are two different comparisons here. The drift checker asks whether published records match the approved configuration. DMARC asks whether a message has an aligned authentication result under its rules. RFC 7489 allows DMARC to pass through aligned SPF or aligned DKIM; requiring both to pass would misstate the protocol. Your deployment can require both mechanisms to be configured as an operational policy, provided the interface explains that distinction.

That separation is especially useful for an AI feature that drafts resident notices. A successful generation, a valid recipient address, and a matching DNS snapshot are different pieces of evidence. Combining them into one optimistic “ready” state makes the eventual failure harder to explain.

## Observe changes without turning uncertainty into absence

The resolver adapter is the next production boundary. Have it return a typed outcome with the queried owner, record type, observation time, resolver context, and either complete answers or a lookup outcome. The example's empty tuple means the adapter observed no relevant record. A timeout must take a different path. Otherwise, a temporary inability to observe DNS becomes a false instruction to recreate a customer's record.

Preserve the evidence.

RFC 7489 describes DNS publication and policy discovery for DMARC, including handling of TXT data and policy selection. A real adapter and validator need to respect that procedure. In particular, checking one exact `_dmarc` owner against a manifest, as the example does, is not a complete implementation of DMARC policy discovery. Label it as an exact-owner drift check. Don't silently present it as the effective policy for every possible From domain.

The intent revision also belongs in every result. Consider a synthetic sequence: revision 7 is collected, a property manager approves revision 8, and the older observation finishes afterward. An unconditional update could mark the new configuration ready using old evidence. Accept the result only against the revision it evaluated, then schedule another observation for the current intent. This is a state-management rule, not an email-standard requirement. Carry the same rule through manual review: if an operator inspects the old snapshot and approves it after the tenant changes the selector, the approval must still refer to revision 7. It cannot authorize the new selector. Store the decision with the manifest revision and observation identifier, and make the activation step check both before changing the tenant state. A retry can reuse a completed observation for its original revision, but a fresh intent needs fresh evidence. This design costs a little more state bookkeeping; it gives support a concrete explanation for why a previously approved configuration is waiting again.

Old evidence stays old.

Cache and retry decisions need the same care. Retain observation timestamps and DNS cache context so operators can distinguish an older answer from a current mismatch. Define a bounded retry policy for inconclusive observations and expose the last useful result alongside the pending check. Repeated uncertainty should remain visible rather than quietly becoming success.

## Make the release gate eval-driven and prompt-cost aware

Turn the sample into a fixture-driven comparison before connecting a collector. Include a matching snapshot, a missing record, an unobserved owner, a changed DMARC policy, multiple TXT answers, and a stale intent revision. Add parser fixtures as you implement actual protocol semantics. The useful assertion is the decision and its reason, not merely that a function returned without an exception.

For notebook-to-prod work, keep fixtures identical across exploratory runs and the deployment check. That makes a parser change reviewable: it should explain which previously flagged snapshots are now accepted and why. Don't invent a benchmark for this. The release evidence is the fixture behavior and the observation attached to the tenant configuration.

An agent can help turn a structured mismatch into a readable explanation for the property manager. Give it the owner, expected value, observed value, and decision reason after deterministic validation. Keep the validation outside the prompt. This is prompt-cost aware by design: there is no reason to spend tokens repeatedly deciding whether two approved record values match, and the explanation can be regenerated only when the evidence changes.

Before enabling a tenant's mail, confirm the approved domain and revision, inspect the observed records, resolve every review item, and check an actual message for authentication and alignment. Record who approved the transition. After release, repeat observations and surface drift with the affected owner and last matching revision. A property manager should be able to tell whether action is needed, which setting changed, and what evidence supports that conclusion.

Keep routing status, intent conformance, and message authentication visible as separate results. That is the operational checklist worth preserving when the notebook becomes a service.

## References

RFC 7489 supplies the DMARC authentication, alignment, DNS publication, policy-discovery, and handling-policy basis used here. The manifest, revision checks, synthetic fixtures, and release workflow are proposed application design choices. The Python example implements an exact snapshot comparison, not SPF evaluation, DKIM signature validation, or a complete DMARC evaluator.

## Further reading

- RFC 7489 — Domain-based Message Authentication, Reporting, and Conformance (DMARC): https://datatracker.ietf.org/doc/html/rfc7489
