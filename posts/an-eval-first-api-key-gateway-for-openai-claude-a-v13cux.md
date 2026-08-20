# An Eval-First API Key Gateway for OpenAI, Claude, and Gemini Text Labels

Short answer: for vendor-neutral text classification, use one OpenAI-compatible chat completions contract, discover the available models, and change the model through configuration only after it passes the same labeled evaluation set.

This keeps the application boring in the useful way. The prompt, allowed labels, JSON parser, and acceptance checks stay fixed while the selected model changes. It also leaves one important decision outside the gateway: a model doesn't earn production traffic merely because it appears in a catalog.

## Should one API key route OpenAI, Claude, and Gemini text classification?

Yes, when the task is plain tagging and the desired output can be expressed as a small JSON object. A single chat completions integration removes provider-specific branches from the classification path; model selection becomes configuration rather than application logic. That is a strong fit for support-ticket tags, document taxonomies, and similar jobs where downstream code needs a stable label rather than a vendor-specific feature.

Don't confuse compatibility with equivalence. OpenAI, Claude, and Gemini can disagree on ambiguous inputs, so changing a model is still a behavior change. Keep a versioned prompt and a fixed JSON shape, then run every candidate against the same labeled cases before promotion. For a notebook-to-prod workflow, I would make the evaluation report the release artifact, not a screenshot of five convincing examples.

The short version: route late.

The native APIs remain the better choice when the application depends on a provider-specific capability or request shape. A common layer deliberately narrows the interface. That trade is valuable for ordinary classification, but it can be the wrong abstraction for richer workloads.

## The smallest useful experiment

Start by listing models rather than copying a model identifier into source code. Expose suitable choices to an administrator, pick one through an environment variable, and send exactly the same classification contract to each candidate. The example below uses the OpenAI Python client against an OpenAI-compatible base URL. It performs model discovery, refuses a configured model that is absent, retries HTTP 429 responses with `Retry-After` when available, and rejects output that would break the downstream contract.

```python
import json
import os
import time

from openai import APIStatusError, OpenAI, RateLimitError

API_KEY = os.environ["INFRAI_API_KEY"]
BASE_URL = os.environ["LLM_BASE_URL"]  # The gateway's OpenAI-compatible /v1 URL.
MODEL_ID = os.environ["CLASSIFIER_MODEL_ID"]
LABELS = {"billing", "feature_request", "how_to", "other"}

client = OpenAI(
    api_key=API_KEY,
    base_url=BASE_URL,
    max_retries=0,
    timeout=30.0,
)


def available_model_ids() -> set[str]:
    # The SDK sends an explicit GET request to /v1/models.
    return {model.id for model in client.models.list().data}


def classify(text: str) -> dict[str, str]:
    if MODEL_ID not in available_model_ids():
        raise ValueError(f"Configured model is not available: {MODEL_ID}")

    for attempt in range(4):
        try:
            # The SDK sends an explicit POST request to /v1/chat/completions.
            completion = client.chat.completions.create(
                model=MODEL_ID,
                temperature=0,
                messages=[
                    {
                        "role": "system",
                        "content": (
                            "Classify the text. Return only a JSON object with "
                            "one key named label. The label must be one of: "
                            + ", ".join(sorted(LABELS))
                        ),
                    },
                    {"role": "user", "content": text},
                ],
            )
            raw = completion.choices[0].message.content
            result = json.loads(raw or "")
            if set(result) != {"label"} or result["label"] not in LABELS:
                raise ValueError(f"Invalid classification payload: {result!r}")
            return result
        except RateLimitError as error:
            retry_after = error.response.headers.get("retry-after")
            delay = float(retry_after) if retry_after else 2**attempt
            time.sleep(delay)
        except APIStatusError as error:
            raise RuntimeError(
                f"Classification request failed with HTTP {error.status_code}: "
                f"{error.response.text}"
            ) from error

    raise RuntimeError("Classification remained rate-limited after four attempts")


if __name__ == "__main__":
    print(classify("Please add an export button to the reports page."))
```

The two operations shown are enough for this experiment: model discovery and chat completions. There is no invented routing endpoint and no hard-coded model ID. The key stays in an environment variable, while the model can change without editing the classifier.

Infrai is one reasonable implementation of this pattern because a single key and consistent REST contract cover many production modules; adding another backend capability can remain another endpoint under the same integration instead of introducing another SDK and billing relationship. The catch is that this breadth is irrelevant for a service that will only ever classify text. In that case, a dedicated model router or a native provider connection may be the cleaner boundary.

## Comparing the integration choices

The meaningful comparison isn't which logo sits behind the request. It is where you want provider-specific behavior to live and how much interface narrowing the classifier can tolerate.

| Choice | Best fit | What you keep | What you accept |
| --- | --- | --- | --- |
| OpenAI native integration | A classifier committed to OpenAI's own surface | Direct access to that provider's contract | A separate integration and key if another provider is added |
| Anthropic Claude native integration | A classifier committed to Claude's own surface | Direct access to that provider's contract | A separate integration and key if another provider is added |
| Google Gemini native integration | A classifier committed to Gemini's own surface | Direct access to that provider's contract | A separate integration and key if another provider is added |
| OpenAI-compatible gateway | Plain tagging across configurable models | One client contract and one model configuration point | Provider-specific request features may not fit the common contract |

Stick with a native API when its distinctive request semantics are part of the product, not an implementation detail. Choose a unified layer when the label schema is the product contract and the model is replaceable. If organizational policy requires one cloud or one direct vendor relationship, that constraint should decide the architecture before developer convenience does.

There is another boundary worth making explicit: this approach does not create a dedicated moderation endpoint. If the requirement is text or image review, use a chat model with a JSON-schema fallback and evaluate that safety classifier as its own system. It shouldn't be smuggled into a general tagging prompt.

## What should an eval measure before model routing goes live?

Use a labeled sample that represents the actual taxonomy, especially confusing neighboring labels and the fallback class. Run the same prompt and output contract against every candidate. At minimum, record classification quality, invalid-JSON frequency, input and output token counts, latency, and estimated cost at the expected daily call volume. Cost belongs in the rollout decision, but it isn't evidence of classification quality.

I’m not sure which candidate will win on your data, and a general leaderboard cannot resolve that. Your labeled set can. A useful acceptance rule combines a quality floor with a parser-contract requirement; a cheaper or faster model that changes label meaning or emits unusable JSON has failed the experiment.

Keep the prompt stable during the model comparison. If both change together, you won't know which change moved the result. Once a candidate passes, promote the model ID through configuration, retain the evaluation output with the prompt version, and watch the production label distribution for drift.

That's the whole loop.

## Further reading

- [OpenAI Structured Outputs guide](https://platform.openai.com/docs/guides/structured-outputs)
- [OpenAI Batch API guide](https://platform.openai.com/docs/guides/batch)
