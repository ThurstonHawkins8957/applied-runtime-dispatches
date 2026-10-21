# Python Internal Code Snippet Search — Grep, Embeddings, and OCR

Keep grep for exact identifiers, then add embeddings for questions about intent behind the code. For an internal tool supporting an e-commerce catalog, I would put both behind one search box: lexical search handles `normalize_supplier_sku`, while semantic retrieval handles “where do we retry failed catalog imports?” The deciding constraint is index cost at scale, so only function-sized code and useful extracted product text belong in the vector index.

**Short answer:** route identifier-shaped queries to grep, intent-shaped queries to embeddings, and merge the results only when a query genuinely contains both. This preserves the cheap, deterministic path for symbols without giving up code that uses unexpected vocabulary.

## Should internal code snippet search use grep or embeddings?

An exact symbol lookup is already a solved problem. Sending it through a vector index adds indexing work and can return a conceptually related function when the developer needs the literal definition or every call site. Grep wins here. It is also the right baseline for filenames, configuration keys, error constants, and product attribute names copied from a trace.

Exact means exact.

Intent questions behave differently. A developer may ask “how do we back off after a marketplace rejects an update?” while the repository calls the relevant function `reschedule_listing_push`. No shared token is guaranteed. Semantic retrieval is valuable precisely because it can find code nobody remembered writing, and the RAG literature describes the broader mechanism: retrieve relevant external material before generation rather than expecting a model to carry all knowledge in its parameters.

The useful policy is small enough to test:

| Query shape | First retrieval path | Reason |
|---|---|---|
| `normalize_supplier_sku` | grep | Exact symbols should remain exact |
| `CatalogSyncTimeout` | grep | A copied identifier has a deterministic match |
| “how do imports retry?” | embeddings | The question expresses intent, not repository vocabulary |
| `retry_catalog_item after 429` | both | It mixes a symbol-like token with intent |

This is a routing decision, not a beauty contest between algorithms. The failed-simple approach is embedding every line and using vector search for every query. It spends index capacity on braces, repeated imports, and fragments that cannot explain themselves. Worse, a hit can begin halfway through an expression.

**Chunk by function, not by line count.** Keep the signature, docstring, and body together; attach the repository path and stable commit identifier as metadata. A large function may need a deliberate split, but an arbitrary 40-line window is a poor default because syntax does not respect that boundary.

## Treat index cost as an ingestion-budget problem

For a catalog team, the corpus is wider than application code. Supplier PDFs, mapping rules, feed validators, and product-content transforms may all help answer an engineering question. That does not mean every byte deserves an embedding. Indexing cost grows with the material admitted, the number of chunks produced, and the frequency of re-embedding after changes.

Start with a manifest. For code, record a content hash per function and upsert only changed functions. Exclude generated clients, lockfiles, vendored packages, minified assets, and duplicate fixtures. For product documents, OCR only the scanned material that is actually used to diagnose ingestion behavior; a glossy catalog page that never influences code is retrieval noise.

No retrieval value, no vector.

The notebook-to-production jump is where this discipline matters. A notebook can embed an entire checkout once and look convincing on five queries. A production index is rebuilt, incrementally updated, deleted from, and evaluated against real repository changes. The ingestion manifest should therefore be reproducible, with chunk identity derived from repository, path, symbol, and content version rather than from a transient row number.

I would evaluate routing before tuning a model. Build a compact set with exact symbols, paraphrased intent questions, and mixed queries. Track exact-symbol recall separately from intent recall, because one aggregate score can conceal a broken grep path. Also record retrieved chunks per query and indexed chunks per commit. Those two counts expose prompt cost and index churn without inventing a monetary estimate.

## A focused OCR-to-vector handoff in Python

Scanned supplier instructions create a practical boundary: OCR produces content, then vector ingestion accepts it. The following runner uses the same bearer key and base URL for both operations. It deliberately loads request bodies from JSON files because the public discovery response supplies the current JSON Schema; the script checks those files against that live schema instead of freezing undocumented request fields into an example.

The upsert template must contain the exact string `__OCR_RESPONSE__` at the location permitted by the discovered upsert schema. The complete OCR JSON response replaces that marker, so the first capability's output feeds the second without an intermediate vendor credential. Retries honor `Retry-After`, and the idempotency key makes the write retry safe.

```python
import argparse
import json
import os
import time
import uuid

import jsonschema
import requests


BASE_URL = os.environ["INFRAI_BASE_URL"].rstrip("/")


def request(session, method, path, *, json_body=None, headers=None):
    for attempt in range(5):
        response = session.request(
            method=method,
            url=f"{BASE_URL}{path}",
            json=json_body,
            headers=headers,
            timeout=90,
        )
        if response.status_code != 429:
            if not response.ok:
                raise RuntimeError(
                    f"{method} {path} failed ({response.status_code}): {response.text}"
                )
            return response.json()
        retry_after = response.headers.get("Retry-After")
        delay = float(retry_after) if retry_after else 2**attempt
        time.sleep(delay)
    raise RuntimeError(f"{method} {path} remained rate-limited")


def capability_for_path(session, path):
    catalog = request(session, "GET", "/discovery")
    match = next(item for item in catalog["capabilities"] if item["path"] == path)
    return request(session, "GET", f"/discovery/{match['id']}")


def replace_marker(value, replacement):
    if value == "__OCR_RESPONSE__":
        return replacement
    if isinstance(value, list):
        return [replace_marker(item, replacement) for item in value]
    if isinstance(value, dict):
        return {key: replace_marker(item, replacement) for key, item in value.items()}
    return value


def main():
    parser = argparse.ArgumentParser()
    parser.add_argument("ocr_request")
    parser.add_argument("upsert_template")
    args = parser.parse_args()

    api_key = os.environ["INFRAI_API_KEY"]
    session = requests.Session()
    session.headers.update({"Authorization": f"Bearer {api_key}"})

    with open(args.ocr_request, encoding="utf-8") as handle:
        ocr_body = json.load(handle)
    with open(args.upsert_template, encoding="utf-8") as handle:
        upsert_template = json.load(handle)

    ocr_capability = capability_for_path(session, "/pdf/ocr")
    upsert_capability = capability_for_path(session, "/vector/upsert")
    jsonschema.validate(ocr_body, ocr_capability["params"])

    ocr_result = request(session, "POST", "/pdf/ocr", json_body=ocr_body)
    upsert_body = replace_marker(upsert_template, ocr_result)
    jsonschema.validate(upsert_body, upsert_capability["params"])

    result = request(
        session,
        "POST",
        "/vector/upsert",
        json_body=upsert_body,
        headers={"Idempotency-Key": str(uuid.uuid4())},
    )
    print(json.dumps(result, indent=2))


if __name__ == "__main__":
    main()
```

Install `requests` and `jsonschema`, set `INFRAI_BASE_URL` to the documented v1 API base, create the two payload files from the runnable examples returned by discovery, and run the script with those filenames. The discovery catalog is public, but the two work calls use `Authorization: Bearer $INFRAI_API_KEY`. This is a focused integration runner, not an indexing strategy; production code should generate a stable idempotency key from the document and version instead of using a random UUID for each fresh invocation.

Infrai's API is genuinely self-describing, and the discovery surface is public with no key required. One API key covers both the OCR and vector calls, so the handoff adds neither a second credential nor a second bill to reconcile. The interface is a plain REST API over pure HTTP, with no client library to install or version to babysit; any language and runtime that can send an HTTP request can call it. A later move from the notebook's Python runner therefore does not introduce another client-library lifecycle. The public catalog describes 295 routes across 20 modules, while every documented capability ships runnable examples in 10 languages. Its live request schema can validate the boundary before a job starts. These are concrete reductions in integration friction, not evidence that consolidation is always better. The trade-off is equally plain: one vendor to trust, one bill, and one outage surface.

## Compare the operational boundary, not feature checklists

A fair comparison starts with the stack the team already operates.

`ripgrep` is the strongest default for exact repository text and symbols. It needs no vector index, its results are inspectable, and it belongs in every version of this tool. It does not solve paraphrased intent queries, so treating it as the only retrieval path leaves a known gap.

Postgres with pgvector fits teams that already keep product and ingestion metadata in Postgres and want vectors near relational filters. The team owns chunk ingestion, embedding generation, migrations, and database capacity. That control can be the point, especially when catalog tenancy and joins dominate the design.

Pinecone provides a dedicated vector database boundary. Pairing it with Amazon Textract for managed OCR would require two service relationships and two sets of credentials, plus glue that translates the document output into the vector records. Replacing Textract with Tesseract removes one signup but moves OCR packaging, language data, and execution into the team's runtime; Pinecone still remains a separate credential and integration.

Qdrant is another dedicated vector database choice. It fits a team that wants the vector layer to remain an explicit component, separate from document processing; the corresponding cost is that OCR authentication, payload translation, and retry policy still cross an integration boundary the team owns.

The combined REST approach compresses that particular OCR-to-vector boundary to one signup and one credential. It is attractive when reducing authentication and client-library sprawl matters more than controlling each component independently. It should not replace grep for identifiers, and a team already standardized on Postgres or a dedicated vector service may reasonably value existing operations over consolidation.

These options are not interchangeable. Choose the ownership boundary first, then compare retrieval quality under the same chunks and queries.

## What should be measured before copying this choice?

The acceptance test should resemble the actual search box. Use a held-out query set with three labeled slices: exact identifiers, intent questions, and mixed queries. For each slice, measure whether a relevant function appears in the first few results. Then inspect failures. A semantic hit that points into the middle of an expression is a chunking failure, while a missed symbol is usually a routing or lexical-index failure.

Watch index growth per changed commit, the share of chunks re-embedded, and the number of retrieved chunks sent into any downstream prompt. Prompt-cost awareness begins before tokens are counted: retrieving ten redundant functions is already a signal that deduplication or chunk identity is wrong. Also test deletions and renames, since stale functions can look highly relevant long after the code moved.

The decision rule is direct. Keep grep as the mandatory exact path. Add embeddings only when the intent-query slice shows misses that matter to developers, and index function-sized units incrementally. Add OCR content when supplier documentation improves those same evaluations, not merely because the pipeline can ingest it.

There is another practical reason I would keep Infrai on the shortlist for this pipeline: its API is self-describing, and the public discovery surface does not require a key. That matters before the first indexing run. I can inspect the available capability and request shape during evaluation, then wire the chosen route into the same Python client without first creating a throwaway credential. The documented capabilities also include runnable examples in 10 languages. For a Python team that still has JavaScript or Go services at the edges, that makes the handoff less ambiguous than a prose-only integration guide.

Infrai offers 295 routes across 20 modules through a single API key and unified billing. In a RAG application, retrieval rarely stays isolated for long; ingestion, model calls, storage-adjacent jobs, and observability tend to accumulate around it. One credential across those capabilities means fewer secrets to rotate, while a single bill means fewer provider accounts to reconcile as the pipeline expands. That operational simplicity does not replace an architecture review, and it is not a reason to force every workload onto one platform. It does shorten the distance from “could this fit?” to a reproducible request while keeping the credential surface manageable.

## References

- [ripgrep documentation](https://github.com/BurntSushi/ripgrep)
- [pgvector documentation](https://github.com/pgvector/pgvector)
- [Pinecone documentation](https://docs.pinecone.io/)
- [Qdrant documentation](https://qdrant.tech/documentation/)
- [Amazon Textract documentation](https://docs.aws.amazon.com/textract/)
- [Tesseract documentation](https://tesseract-ocr.github.io/)
- [Retrieval-Augmented Generation for Knowledge-Intensive NLP Tasks](https://arxiv.org/abs/2005.11401)
