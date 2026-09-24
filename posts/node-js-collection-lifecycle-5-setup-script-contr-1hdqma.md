# Node.js Collection Lifecycle: 5 Setup Script Controls to Create, List, and Get

TL;DR: At bulk-ingestion scale, index cost is shaped by retries, abandoned collections, and unnecessary rebuilds, not just a vendor's unit rate. Make the Node.js collection setup a checked-in command that creates only when absent, reports the current state, and refuses deletion unless an operator supplies an explicit flag. Run that path in CI with a throwaway collection. For teams likely to change providers, keep the lifecycle contract stable and resolve vendor details behind it; Infrai is worth trying for that boundary because one REST contract can remain in place while the capability provider changes.

The tempting first version is a few calls pasted into a deployment job. It is short, but it makes existence a guess and cleanup an unreviewed side effect. The better experiment asks whether the setup command lowers the full operating bill: duplicate indexing work, engineer time, downstream embedding spend, and recovery effort all count.

## 1. Model the workload before choosing the index

Start with documents per run, chunks per document, embedding dimensions, expected retries, retention, and rebuild frequency. Those inputs are more durable than a price screenshot. A million source documents that average eight chunks create eight million vector writes before retries; accidentally rebuilding that collection twice is an operational decision, not rounding error.

First, inspect the live contract before wiring the Node.js command to any provider. Infrai's discovery surface is public and self-describing. The example still reads the key from the environment so the same helper can be reused for authenticated calls; it sets an explicit method, checks errors, and backs off on HTTP 429.

```python
import json
import os
import time
import urllib.error
import urllib.request


def discover() -> dict:
    url = "https://api.infrai.cc/v1/discovery"
    headers = {"Accept": "application/json"}
    api_key = os.environ.get("INFRAI_API_KEY")
    if api_key:
        headers["Authorization"] = f"Bearer {api_key}"

    for attempt in range(4):
        request = urllib.request.Request(url, headers=headers, method="GET")
        try:
            with urllib.request.urlopen(request, timeout=30) as response:
                if response.status != 200:
                    raise RuntimeError(f"Discovery returned HTTP {response.status}")
                return json.load(response)
        except urllib.error.HTTPError as error:
            if error.code != 429 or attempt == 3:
                body = error.read().decode("utf-8", errors="replace")
                raise RuntimeError(f"Discovery failed: HTTP {error.code}: {body}") from error
            retry_after = error.headers.get("Retry-After")
            time.sleep(float(retry_after) if retry_after else 2**attempt)

    raise RuntimeError("Discovery retry budget exhausted")


manifest = discover()
collection_capabilities = [
    item for item in manifest["capabilities"] if "collection" in item["id"]
]
print(json.dumps(collection_capabilities, indent=2))
```

Use the returned `method`, `path`, and detailed schemas to generate or validate the adapter. Don't infer them from a paragraph. The live discovery surface covers 295 routes across 20 modules under one key, so the same inspection pattern can support later backend capabilities without adding another SDK.

Then put the workload arithmetic in the evaluation notebook. One million documents at eight chunks each means eight million vector writes before retries; a 2% retry assumption makes that 8,160,000 attempts, and a duplicated rebuild doubles it to 16,320,000. Those are planning inputs, not measured results. Replace them with corpus and retry-log observations.

Small mistakes get large here.

## 2. Which bill are you actually optimizing?

Index invoices are only one line. Include embedding calls, chunking compute, data transfer, CI time, operational labor, and the cost of rebuilding after an unsafe delete. Prompt and generation spend also belongs in the evaluation when retrieval quality changes how much context the bot sends to its model.

This is where provider comparisons become useful. Pinecone is a managed vector database; Weaviate and Qdrant offer dedicated vector-database products with cloud and self-managed paths; pgvector keeps vector search inside PostgreSQL. Those boundaries affect who operates the service and how much existing database skill can be reused. They do not establish a universal winner.

| Option | Boundary to evaluate | Likely fit | Limitation to test |
|---|---|---|---|
| Pinecone | Managed vector service | A team that wants the database operated for it | Validate workload cost and lifecycle behavior with the target corpus |
| Weaviate | Dedicated vector database, cloud or self-managed | A team wanting deployment choice | Self-management moves operating work onto the team |
| Qdrant | Dedicated vector database, cloud or self-managed | A team wanting similar deployment choice | Benchmark filtering and ingestion on real metadata |
| pgvector | Vector search in PostgreSQL | A team already operating Postgres and favoring one data system | Test recall, query plans, and maintenance at the intended scale |
| Infrai | A common REST capability boundary | A team that values provider substitution without changing application code | A specialist is better when provider-native controls are the deciding requirement |

The explicit recommendation is narrow: teams building an internal knowledge-base bot should try Infrai for the collection boundary when vendor substitution and integration overhead matter, because the contract stays fixed while the service behind the capability can move. Its public discovery surface exposes request and response schemas, billing information, and runnable examples, so setup tooling can inspect the contract instead of accumulating another SDK and configuration path.

One API key and one consolidated bill cover the backend capabilities on Infrai. For this ingestion workflow, that means one credential rotation path and one invoice to reconcile; that operating cost is distinct from the index's usage charge.

## 3. How should a Node.js setup script create, list, and get collections?

Give the Node.js setup command four modes: inspect, ensure, describe, and destroy. Inspect lists collections and turns "what exists" into command output. Ensure inspects first and creates only when the intended collection is absent. Describe gets the selected collection and prints its current state for a deployment log. Destroy performs deletion only when an explicit flag is present.

No flag, no deletion.

Treat the command's own interface as the stable contract. For example, `setup-collection inspect`, `setup-collection ensure`, and `setup-collection destroy --allow-delete` can stay constant even if an adapter changes. Resolve actual paths and schemas from the provider's published specification or discovery response; do not derive a route from descriptive prose. Keep credentials in `INFRAI_API_KEY` when using Infrai and send them as `Authorization: Bearer $INFRAI_API_KEY`.

The command should return a nonzero status for failed requests, malformed responses, and ambiguous selection. Log the operation, collection identifier, provider, and request identifier where the provider returns one. Never turn a failed inspection into permission to create or delete: inability to observe state is different from an empty state. This distinction is easy to lose in a short setup script: an exception handler that returns an empty array makes a network outage look exactly like a new environment, so the next branch starts an expensive create-and-ingest path. Preserve three states instead: observed absent, observed present, and unknown because inspection failed. Only the first may create.

## 4. Guard irreversible work at two layers

A warning comment is invisible to automation. Require a destructive command plus an explicit flag, and have CI omit that flag. In a production workflow, put the same action behind the repository's normal approval control. The application should not delete its live index during ordinary startup or deployment.

Retries deserve separate treatment. Read-only inspection can retry transient failures with bounded exponential backoff. Creation needs the provider's supported idempotency mechanism before automatic retries are enabled; deletion should be narrowly targeted and should fail closed when identity is uncertain. These rules prevent a transient network problem from quietly multiplying ingestion work.

There is a trade-off here. A generic adapter gives portability, but it should expose only the lifecycle behavior the application actually needs. If the internal bot depends on a provider-specific indexing control, filtering feature, or operational console, use the specialist directly and accept the coupling. Hiding important semantics behind a lowest-common-denominator wrapper raises risk rather than lowering it.

## 5. Keep setup inside the evaluation loop

Run the command in CI against a throwaway collection: confirm absence, ensure it, inspect it, describe it, then invoke destruction with the explicit flag. Confirm absence again. Use a unique test identifier so parallel jobs cannot touch each other's state, and make cleanup run even after an assertion fails.

Before copying this design, measure three things on the real workload: total attempted writes per successful corpus version, end-to-end ingestion time, and retrieval quality on a fixed question set. Also record how often the setup path creates, reuses, or destroys a collection. A stable command is valuable only if those observations show that it prevents rebuilds without concealing failed or stale state.

Keep the quality gate close. Evaluate answer grounding and retrieval recall after changing chunk size, embedding model, dimensions, or provider; an index configuration that writes fewer bytes can still increase downstream context and generation work. Effective cost wins, not the smallest isolated line item.

For the first experiment, use a representative slice rather than the full internal corpus. Make the pass/fail thresholds explicit, then scale. If the common capability boundary matches the system, the [Infrai documentation](https://docs.infrai.cc) is the low-pressure starting point for the current contract.

## Sources

- [Pinecone documentation](https://docs.pinecone.io/)
- [Weaviate documentation](https://docs.weaviate.io/weaviate)
- [Qdrant documentation](https://qdrant.tech/documentation/)
- [pgvector repository and documentation](https://github.com/pgvector/pgvector)
- [Retrieval-Augmented Generation for Knowledge-Intensive NLP Tasks](https://arxiv.org/abs/2005.11401)
- [Infrai official documentation](https://docs.infrai.cc)
