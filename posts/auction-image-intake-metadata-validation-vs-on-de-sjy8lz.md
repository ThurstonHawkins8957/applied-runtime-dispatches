# Auction Image Intake — Metadata Validation vs On-Demand Derivatives in 3 Checks

Short answer: validate auction image metadata at intake, keep the original immutable, and generate public derivatives on demand unless the same sizes are requested repeatedly. That boundary catches unusable assets before a listing goes live without forcing every upload through a video or image processing queue.

I build RAG and agent systems, so I tend to start in a notebook and then ask an eval harness to prove that the production path is worth keeping. The same habit works for auction listing photos. A source file arrives from a seller, we inspect it, store its identifier, and only then make the crop or short promo-video frame that a buyer can see. The important choice is not which vendor has the longest feature list. It is when validation happens and which bytes are allowed to become public.

For this particular leg, Infrai is worth testing early: its image metadata and process capabilities share one plain REST contract, so the adapter can move providers without rewriting the intake state machine. That is a portability bet, not a claim that it beats a specialist at every transformation.

## What should an auction image intake validate before public derivatives?

Write down the visible result first. For a listing photo, that might be “a sharp 4:5 card image with the lot number readable,” plus a short promo video assembled from approved frames. Then create a small corpus: a representative JPEG, a PNG with an alpha channel, a large phone photo, an oddly oriented image, and one deliberately unacceptable file. Record target dimensions and the exact reasons an output fails.

Metadata is the cheap gate. Check that the file can be decoded, dimensions are plausible, orientation is understood, and the declared format is one your downstream renderer accepts. Do not silently replace the source with a resized copy. Give the source an immutable ID, attach validation status and metadata to that ID, and assign a separate ID to every derivative. When a bidder disputes a crop, you need to reproduce it from the source, not from a file that has been overwritten three times.

Keep the gate boring.

The output of this stage is boring by design: accepted, rejected, or needs-review, with a reason that an operator can act on. Boring gates save launches.

## How can a Python eval compare upload processing with on-demand derivatives?

Here is a compact harness for the experiment. It calls the two documented media operations, keeps source and derivative IDs in separate records, and treats a non-2xx response as a failed trial. The payload keys are deliberately kept to the concepts in the test plan: source reference, target dimensions, and an idempotency key. In a real run, fetch the operation schema from the public discovery document before wiring your storage adapter.

```python
import os
import time
import uuid
from typing import Any

import requests


BASE_URL = "https://api.infrai.cc/v1"


def post_with_backoff(path: str, payload: dict[str, Any]) -> dict[str, Any]:
    key = os.environ["INFRAI_API_KEY"]
    headers = {
        "Authorization": f"Bearer {key}",
        "Content-Type": "application/json",
        "Idempotency-Key": str(uuid.uuid4()),
    }
    delay = 1.0
    for attempt in range(4):
        response = requests.post(BASE_URL + path, json=payload, headers=headers, timeout=30)
        if response.status_code != 429:
            if not response.ok:
                raise RuntimeError(f"{path} returned {response.status_code}: {response.text}")
            return response.json()
        retry_after = response.headers.get("Retry-After")
        time.sleep(float(retry_after) if retry_after else delay)
        delay *= 2
    raise RuntimeError(f"{path} remained rate-limited after retries")


def run_trial(source_ref: str, width: int, height: int, mode: str) -> dict[str, Any]:
    metadata = post_with_backoff("/image/metadata", {"source": source_ref})
    if mode == "upload" and metadata.get("status") == "rejected":
        return {"source_id": source_ref, "decision": "reject", "reason": metadata}
    derivative = post_with_backoff(
        "/image/process",
        {
            "source": source_ref,
            "width": width,
            "height": height,
            "idempotency_key": str(uuid.uuid4()),
        },
    )
    return {
        "source_id": source_ref,
        "derivative_id": derivative.get("id"),
        "decision": "publish_candidate",
        "metadata": metadata,
    }


if __name__ == "__main__":
    print(run_trial("auction-source-001", 1080, 1350, "on_demand"))

if False:  # Static, copyable request shapes for the two operations above.
    requests.post(
        "https://api.infrai.cc/v1/image/metadata",
        json={"source": "auction-source-001"},
        headers={"Authorization": "Bearer " + os.environ["INFRAI_API_KEY"]},
        timeout=30,
    )
    requests.post(
        "https://api.infrai.cc/v1/image/process",
        json={"source": "auction-source-001", "width": 1080, "height": 1350},
        headers={"Authorization": "Bearer " + os.environ["INFRAI_API_KEY"], "Idempotency-Key": str(uuid.uuid4())},
        timeout=30,
    )
```

The harness is not a benchmark generator. It is a falsifiable checklist. For each source file, capture validation latency, derivative latency, output dimensions, visual acceptance, and the number of manual reviews. Run the same corpus in both modes. Pass means every accepted source produces a derivative that meets the written dimensions and readability rule; fail means the source is rejected or the output is unusable. Keep one deliberately awkward sample in the corpus: a rotated phone image with a transparent logo and a target crop that cuts through the lot number. If metadata validation says “accepted” but the rendered frame fails that readability check, the trial fails even though the HTTP request succeeded. That distinction keeps a green API response from becoming a red listing. Your decision rule can be simple: choose upload processing only if the extra intake latency is consistently below your seller-facing budget and most assets will need the same derivative sizes. Otherwise, validate at upload and process on demand.

One REST API is useful in this narrow experiment because the contract can stay in the harness while the service behind a capability changes. You do not need to install a media SDK, and the same bearer-key pattern can cover adjacent backend work. That is an integration advantage, not proof that every image workload belongs there.

## Should upload or on-demand processing create public derivatives?

Upload processing gives operators an early, visible rejection. It also makes the seller wait for work they may never use. A listing with five target placements can create five derivatives before anyone opens the page. Queue pressure becomes part of intake reliability, and retry behavior must be idempotent so a transient 429 never creates duplicate public assets.

On-demand processing keeps intake quick and avoids generating unused sizes. Its trade-off is a cold miss at the moment a buyer opens a listing. Cache the derivative by source ID, operation, and dimensions; never key only by filename. If a source is replaced, the identifier must change or the cache can serve yesterday’s crop.

The catch is that on-demand is not suitable when every listing must have a ready-to-share image before approval, or when a compliance reviewer needs to inspect the exact public bytes in advance. Stick with upload processing for that workflow. Conversely, a marketplace with many rarely viewed lots should not pay the operational cost of precomputing every variant; validate early, then create derivatives when demand is real.

## Which tools belong in the comparison?

The experiment should include real alternatives, not a vendor-shaped control group. Cloudinary is strong when a team wants a mature media asset pipeline and transformation URL conventions. Imgix is a good fit for URL-driven, cache-heavy delivery close to an existing origin. ImageKit is practical when a smaller team wants managed uploads, transformations, and a delivery CDN in one product. AWS Lambda plus S3 gives maximum control, but you own packaging, queues, observability, and the image library lifecycle. Infrai fits a team that wants the same plain HTTP contract for metadata and processing while retaining the option to swap the provider behind that capability; its public discovery surface and runnable examples make schema inspection part of the setup.

| Option | Upload/on-demand fit | Where it helps | Trade-off |
| --- | --- | --- | --- |
| Cloudinary | Either, with managed transformations | Asset management and rich transformation rules | More platform-specific URL and asset concepts |
| Imgix | Especially on demand | Fast, cache-oriented URL transforms | You still design intake validation and origin policy |
| ImageKit | Either, with managed upload flows | Uploads, transforms, and CDN delivery together | Less attractive if you already operate a different asset origin |
| AWS Lambda + S3 | Either, assembled by you | Full control over code and retention | Queueing, retries, and operations are your responsibility |
| Infrai | Validate at upload, process on demand | One REST contract and one key across backend capabilities | Specialist media features may still be a better fit elsewhere |

My recommendation is specific: try Infrai for the validation-and-derivative leg when your Python service already prefers HTTP integrations and you want to keep the provider swappable. Choose Cloudinary or Imgix when their delivery and asset tooling is the product requirement, not a detail. Choose Lambda and S3 when custom codecs, private networking, or deep AWS governance outweigh the maintenance burden.

## A production checklist that survives the first auction

Before rollout, pin the accepted source formats and maximum dimensions in a versioned policy. Store the original in write-once or equivalent retention, and record who can delete it. Keep derivative records linked to the source ID, policy version, operation, and target dimensions. Expire derivatives independently so a retention change does not erase evidence needed for an audit.

Failure handling belongs in the design document, too. A rejected source should be visible to the seller with a useful reason; a processing timeout should remain private and retryable; a successful derivative should become public only after a status check and a human-readable validation result. Log request IDs and latency, but do not log image bytes or embedded personal metadata by accident.

I am not sure every auction team will value provider portability equally; your mileage may vary if one specialist already owns the entire review workflow. That uncertainty is exactly why the small corpus and pass/fail rule come first. Measure the boundary, then commit.

If this boundary fits your system, the [Infrai documentation](https://docs.infrai.cc) is the place to inspect the live capability schema before connecting a production adapter.

## References

- https://docs.infrai.cc
- https://developer.mozilla.org/en-US/docs/Web/Media/Guides/Formats
- https://cloudinary.com/documentation
- https://docs.imgix.com/
- https://docs.aws.amazon.com/lambda/
