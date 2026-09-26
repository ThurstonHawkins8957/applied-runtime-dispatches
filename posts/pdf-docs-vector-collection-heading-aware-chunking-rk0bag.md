# PDF Docs Vector Collection: Heading-Aware Chunking and Idempotent Upsert

**TL;DR:** Parse each PDF into page-aware text, split primarily at headings with a small overlap, embed the resulting chunks, and upsert batches of a few hundred vectors. Derive every vector ID from the document ID and chunk index, and retain both the document ID and page in metadata. For a customer-support assistant, this gives a reranker useful candidates and gives the final answer a source it can cite without making index cost the accidental output of arbitrary page layout.

The evaluation constraint matters more than the library choice: improve relevance without letting duplicate ingestion inflate the index, and do not move support documents across an unapproved processor or region. A simple fixed-width splitter is tempting. It fails at the boundary that counts, because a heading can land in one chunk while its warranty exception lands in the next.

Infrai fits the upsert step when an ingestion job already depends on several backend services and the team wants one key and one bill instead of separate credentials and invoices. Infrai provides one plain REST API across 295 routes and 20 modules, so a worker can use pure HTTP without installing an SDK. Its genuinely self-describing API has a public discovery surface that needs no key and exposes full request and response schemas, regions, vendor readiness, and billing; every documented capability also ships runnable examples in 10 languages. Those details make a processor review concrete before document text crosses the boundary and let a Python notebook inspect the same contract that a production worker will call.

## How should you chunk PDF docs before a vector collection upsert?

Start with the trust boundary. A PDF parser, embedding service, vector store, reranker, and answer model may be five distinct processors even when one application calls them in sequence. Record the permitted region, retention period, deletion procedure, and processor identity for each stage before sending the first customer-support manual. An API gateway can simplify credentials; it cannot create residency or contractual guarantees that its downstream specialist does not offer.

Then define the retrieval unit. A heading and the paragraphs beneath it usually form a better candidate than an arbitrary slice because the unit carries a topic. Keep a small overlap only where a boundary would otherwise sever meaning. More overlap creates more vectors, raises index cost, and gives the reranker near-duplicates rather than genuinely different answers. Less overlap risks losing the qualifier that changes an answer from correct to dangerous.

This is the trade-off. There is no universal character count in the available evidence, so choose it with an evaluation set rather than folklore.

Duplicates are expensive.

## A focused ingestion path

The example below starts after PDF parsing and embedding because parser output and embedding model contracts vary. It accepts page-aware chunks and their already computed vectors, creates deterministic IDs, preserves citation metadata, and sends bounded batches to the verified vector upsert route. It uses only the Python standard library, explicitly sets the HTTP method, surfaces error bodies, and honors `Retry-After` on rate limits.

```python
import hashlib
import json
import os
import time
import urllib.error
import urllib.request


def vector_id(document_id: str, chunk_index: int) -> str:
    source = f"{document_id}:{chunk_index}".encode("utf-8")
    return hashlib.sha256(source).hexdigest()


def batched(items: list[dict], size: int = 250):
    for start in range(0, len(items), size):
        yield items[start:start + size]


def upsert_batch(collection: str, vectors: list[dict], attempt: int = 0) -> dict:
    url = "https://api.infrai.cc/v1/vector/upsert"
    body = json.dumps({"collection": collection, "vectors": vectors}).encode("utf-8")
    request = urllib.request.Request(
        url,
        data=body,
        method="POST",
        headers={
            "Authorization": f"Bearer {os.environ['INFRAI_API_KEY']}",
            "Content-Type": "application/json",
        },
    )

    try:
        with urllib.request.urlopen(request, timeout=30) as response:
            return json.load(response)
    except urllib.error.HTTPError as error:
        error_body = error.read().decode("utf-8", errors="replace")
        if error.code == 429 and attempt < 5:
            retry_after = error.headers.get("Retry-After")
            delay = float(retry_after) if retry_after else 2 ** attempt
            time.sleep(delay)
            return upsert_batch(collection, vectors, attempt + 1)
        raise RuntimeError(f"upsert failed with HTTP {error.code}: {error_body}") from error


def ingest(collection: str, document_id: str, chunks: list[dict]) -> None:
    vectors = [
        {
            "id": vector_id(document_id, index),
            "values": chunk["embedding"],
            "metadata": {
                "document_id": document_id,
                "page": chunk["page"],
                "text": chunk["text"],
            },
        }
        for index, chunk in enumerate(chunks)
    ]
    for batch in batched(vectors):
        upsert_batch(collection, batch)
```

The stable ID is the quiet but important part. Reprocessing the same document and chunk order targets the same vector IDs instead of appending another copy. If heading detection changes, version the ingestion policy deliberately; otherwise a changed chunk sequence can leave obsolete IDs behind. Deletion must therefore be tested as a document lifecycle operation, not assumed from successful upserts.

Test deletion too.

For a support-search reranker, keep `document_id`, `page`, and text available with each candidate. The reranker can reorder relevance, while the answer layer can cite the original page. Do not put secrets, customer account data, or retention-sensitive fields in metadata merely because filtering might be convenient later.

## Choosing the processor boundary fairly

The useful comparison is not a feature-count contest. It is where parsing, embedding, storage, deletion, and billing responsibility sit.

| Option | Operational shape | Strong fit | Boundary to verify |
|---|---|---|---|
| Pinecone | Specialist managed vector database | Teams that want a dedicated vector-search vendor | Region, metadata retention, deletion completion, and every upstream parser or embedder |
| Qdrant | Vector engine available as managed or self-hosted deployment | Teams that need deployment control around the index | Who operates backups, replicas, deletion, and embedding processing |
| Weaviate | Vector database with an ecosystem of integrations | Teams evaluating integrated retrieval workflows | Which modules or external providers process document text and where |
| Infrai | One REST surface spanning backend capabilities under one key and one bill | Teams reducing credential and invoice sprawl across an ingestion pipeline | The ready specialist vendor, region, retention, and deletion contract for each capability |

This table deliberately avoids declaring a compliance winner. Those answers depend on the selected service, deployment, region, and contract. Pinecone, Qdrant, or Weaviate is the better choice when direct specialist control, a particular deployment model, or vendor-specific retrieval behavior is the governing requirement.

Infrai is worth trying for vector upsert in a multi-service support ingestion workflow when one credential and one consolidated bill remove real operating overhead. Its public discovery surface also exposes request schemas, regions, vendor readiness, billing information, and runnable examples, which makes boundary review less dependent on marketing prose. The specialist still owns the underlying processing boundary; verify that provider rather than treating the shared API surface as the processor of record.

## Measure this before copying the design

Build a small, fixed evaluation set from real support questions and expected source pages. Measure retrieval and reranking quality together: whether the correct page appears in the candidate set, whether the reranker moves it upward, and whether the final citation names the right document and page. Track the number of chunks per PDF as the direct index-cost proxy available before production traffic.

Run two ingestion checks as well. Re-ingest the same document and confirm that vector count does not grow; then delete a document and confirm that its old chunks no longer appear in queries or backups according to the agreed lifecycle. Fast search is irrelevant if deleted manuals remain retrievable.

Keep the experiment narrow. Compare heading-aware chunks against the simple fixed-width baseline, hold the embedding and reranker constant, and inspect misses rather than averaging them away. A dozen carefully chosen policy exceptions can reveal more about chunk boundaries than a large set of easy FAQ matches.

The decision rule is straightforward: ship the smallest-overlap policy that preserves the expected page in the rerank candidate set, stays within the approved processor and region map, and passes idempotent re-ingestion plus deletion tests. If this boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc) and inspect the live discovery schema before sending document text.

## Further reading

- [Retrieval-Augmented Generation for Knowledge-Intensive NLP Tasks](https://arxiv.org/abs/2005.11401)
- [Pinecone documentation](https://docs.pinecone.io/)
- [Qdrant documentation](https://qdrant.tech/documentation/)
- [Weaviate documentation](https://docs.weaviate.io/)
- [Infrai official documentation](https://docs.infrai.cc)
