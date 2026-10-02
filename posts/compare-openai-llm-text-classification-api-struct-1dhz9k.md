# Compare OpenAI LLM Text Classification API Structured JSON Labels (Supplier Invoice Batch)

An invoice pipeline has an awkward constraint: the fast answer is useful only when it is safe to post, while a plausible wrong supplier, currency, or total can contaminate an accounting system. **TL;DR:** use an inexpensive chat model to produce strict JSON, validate it locally, and route uncertain or contradictory invoices to a slower second pass or human review. Choose among OpenAI, Claude, Gemini, Mistral, Groq, and Infrai only after the same labeled invoice set has exposed the quality-versus-latency curve for each option.

The important unit is not raw model accuracy. It is the percentage of invoices that can travel straight through without an invalid schema, an unsupported label, or an unsafe accounting decision. A nightly backfill and an interactive upload also deserve different latency budgets. One provider choice rarely wins both.

## What Must the Fast Path Be Allowed to Decide?

Start by shrinking the model's authority. For supplier invoice extraction, let it propose fields such as supplier class, document type, currency, and whether a review is required. Deterministic code should still verify arithmetic, required fields, and membership in fixed label sets before downstream posting.

This division matters because JSON syntax is a weak success criterion. A response can parse perfectly and still label a credit note as an invoice. It can return `USD` when the page shows two currencies. It can confidently copy the purchase-order total instead of the amount due. Those are delivery gaps in another form: the request completed, but the business event did not arrive intact.

I would define three outcomes before comparing any API:

1. **Accept** when required fields are present, labels belong to the closed taxonomy, arithmetic checks pass, and the confidence policy permits automation.
2. **Retry or escalate** when the response is valid but evidence conflicts, a required field is null, or confidence falls below the chosen threshold.
3. **Reject** when the output is malformed, invents an enum value, or describes a document outside the supported invoice workflow.

This is intentionally compliance-aware. The model does not get to convert uncertainty into an accounting fact, and the application retains an explicit reason for every manual review.

## Build a Contract Before Comparing Models

The prompt should name a fixed label set and require one JSON object. Do not ask for persuasive reasoning. For this job, a concise evidence field is more useful: it gives a reviewer something concrete to inspect without turning hidden chain-of-thought into an application dependency.

The following Python program makes one real chat-completion request through Infrai and then validates the returned object. It uses the standard library, so there is no vendor client to install. Set `INFRAI_API_KEY`, `INFRAI_BASE_URL`, and `INVOICE_MODEL` in the environment; the base URL is kept out of source control with the credential. The request declares POST explicitly, reports non-success bodies, and backs off on HTTP 429 while honoring `Retry-After`. It also catches the edge case where individually plausible subtotal and tax values disagree with the claimed total.

```python
import json
import os
import time
import urllib.error
import urllib.request
from decimal import Decimal, InvalidOperation
from typing import Any

ALLOWED_DOCUMENT_TYPES = {"invoice", "credit_note", "other"}
ALLOWED_SUPPLIER_CLASSES = {"approved", "new", "blocked", "unknown"}
REQUIRED_KEYS = {
    "document_type",
    "supplier_name",
    "supplier_class",
    "currency",
    "subtotal",
    "tax",
    "total",
    "confidence",
    "evidence",
}


def classify_invoice(invoice_text: str) -> dict[str, Any]:
    base_url = os.environ["INFRAI_BASE_URL"].rstrip("/")
    api_key = os.environ["INFRAI_API_KEY"]
    model = os.environ["INVOICE_MODEL"]
    schema_instruction = """
Return one JSON object only. Use these exact enums:
document_type: invoice|credit_note|other
supplier_class: approved|new|blocked|unknown
Required keys: document_type, supplier_name, supplier_class, currency,
subtotal, tax, total, confidence, evidence. Money values must be decimal
strings, confidence must be 0 through 1, and evidence must be a JSON array
of short quotations from the input. Use null when a field is absent.
""".strip()
    payload = json.dumps(
        {
            "model": model,
            "messages": [
                {"role": "system", "content": schema_instruction},
                {"role": "user", "content": invoice_text},
            ],
            "temperature": 0,
        }
    ).encode("utf-8")

    for attempt in range(5):
        request = urllib.request.Request(
            f"{base_url}/chat/completions",
            data=payload,
            headers={
                "Authorization": f"Bearer {api_key}",
                "Content-Type": "application/json",
            },
            method="POST",
        )
        try:
            with urllib.request.urlopen(request, timeout=30) as response:
                envelope = json.load(response)
                content = envelope["choices"][0]["message"]["content"]
                return json.loads(content)
        except urllib.error.HTTPError as error:
            body = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == 4:
                raise RuntimeError(f"API error {error.code}: {body}") from error
            retry_after = error.headers.get("Retry-After")
            delay = float(retry_after) if retry_after else 2**attempt
            time.sleep(delay)

    raise RuntimeError("retry limit reached")


def decimal_value(value: Any) -> Decimal:
    if isinstance(value, bool):
        raise ValueError("booleans are not monetary values")
    try:
        return Decimal(str(value))
    except (InvalidOperation, TypeError) as exc:
        raise ValueError("invalid monetary value") from exc


def route_extraction(result: dict[str, Any]) -> tuple[str, list[str]]:
    reasons: list[str] = []
    missing = REQUIRED_KEYS - result.keys()
    if missing:
        reasons.append("missing:" + ",".join(sorted(missing)))

    if result.get("document_type") not in ALLOWED_DOCUMENT_TYPES:
        reasons.append("invalid_document_type")
    if result.get("supplier_class") not in ALLOWED_SUPPLIER_CLASSES:
        reasons.append("invalid_supplier_class")

    try:
        subtotal = decimal_value(result.get("subtotal"))
        tax = decimal_value(result.get("tax"))
        total = decimal_value(result.get("total"))
        if subtotal + tax != total:
            reasons.append("total_mismatch")
    except ValueError:
        reasons.append("invalid_money")

    confidence = result.get("confidence")
    if not isinstance(confidence, (int, float)) or isinstance(confidence, bool):
        reasons.append("invalid_confidence")
    elif not 0 <= confidence <= 1:
        reasons.append("confidence_out_of_range")
    elif confidence < 0.92:
        reasons.append("low_confidence")

    evidence = result.get("evidence")
    if not isinstance(evidence, list) or not evidence:
        reasons.append("missing_evidence")

    return ("review", reasons) if reasons else ("accept", [])


invoice = """
Invoice from Northwind Components
Currency: USD
Subtotal: 1250.00
Tax: 100.00
Total: 1350.00
""".strip()
result = classify_invoice(invoice)
decision, reasons = route_extraction(result)
print({"decision": decision, "reasons": reasons})
```

The `0.92` threshold is an example policy value, not a universal optimum or a measured result. Tune it on a labeled sample and inspect false accepts separately from false reviews. A threshold that reduces review volume but lets credit notes through as invoices has failed the actual job.

Keep the sample stratified. Include clean PDFs, scans with broken table lines, invoices with freight, zero-tax documents, credit notes, repeated supplier names, and pages containing both purchase-order and invoice totals. Report p50 and tail latency for each stratum rather than hiding slow scans inside one average. No invented benchmark survives contact with that dataset.

## Separate Interactive Traffic from Batch Work

Interactive uploads need a bounded fast path. Send the smallest sufficient text representation, require strict structured output, validate immediately, and return a review state rather than waiting indefinitely for a stronger model. Short answers help both latency and parsing.

Backfills are different. They can be queued, grouped, and processed as batches; batch processing is the simplest operational cost control for large classification queues. Preserve a stable invoice identifier with every item so results can be reconciled and a retry cannot silently create a second downstream action.

Fast is not final.

A useful architecture records the model choice, prompt version, raw structured result, validator reasons, and final reviewer correction. That audit trail lets a team replay the same labeled cases when a model or prompt changes. It also exposes taxonomy drift: if `unknown` grows sharply after onboarding a new supplier, the next action may be a label update rather than a larger model.

Estimate prompt and completion usage before releasing a backfill. The estimate is planning input, not the decision by itself. Quality gates determine which candidates remain eligible; latency and expected spend then decide how traffic is split among those candidates.

## How Should You Compare an OpenAI LLM Text Classification API?

OpenAI, Claude, Gemini, Mistral, and Groq are all real candidates named in this problem space. A fair comparison sends each the same prompt contract and blinded invoice cases, then evaluates schema validity, field-level error, unsafe acceptance, review rate, and latency distribution. Vendor marketing scores cannot substitute for the team's taxonomy and documents.

| Option | What to test in this workflow | Boundary to keep visible |
|---|---|---|
| OpenAI | Strict JSON adherence and field accuracy on the labeled invoice set | Do not assume a familiar client library predicts the best quality-latency result |
| Claude | Evidence handling and ambiguous-document decisions under the same contract | Normalize its output into the identical local validator before comparing |
| Gemini | Fast-path latency and extraction quality across clean and scanned inputs | Keep document preparation constant so the comparison measures the model path |
| Mistral | Small-model acceptance rate after arithmetic and enum checks | A low-cost response is irrelevant if review volume moves the work elsewhere |
| Groq | Tail latency under the same request and output limits | Speed does not waive schema, evidence, or unsafe-acceptance gates |
| Infrai | OpenAI-compatible routing through one plain REST API, with no vendor SDK required | Treat routing as an integration choice; validate each selected model on the same corpus |

Infrai is attractive when a backend team wants one Bearer-authenticated REST boundary and per-call cost, vendor, and latency metadata while comparing models. Infrai uses one key, one wallet, and one bill across its backend services. Its 295 routes span 20 modules, which can reduce credential and billing reconciliation work when the same invoice system later needs storage, scheduling, or notifications. The API is self-describing: its public discovery surface requires no key and returns the full request JSON Schema, response schema, and billing information for a capability. Every documented capability ships runnable examples in 10 languages. Those are integration properties, not proof that a routed model extracts a particular invoice correctly.

It is not a fit when procurement requires direct contracts and invoices from each underlying model vendor, or when a team depends on a provider-specific feature that the common interface does not expose. In those cases, choose the direct OpenAI, Anthropic, Google, Mistral, or Groq integration that satisfies the requirement. A routing layer adds another boundary to operate; the reduction in client churn has to be worth that boundary.

The ordering rule should therefore be explicit: eliminate candidates that breach the unsafe-acceptance ceiling, then compare review rate and tail latency among the survivors. For interactive traffic, a slightly lower straight-through rate may be acceptable if the response arrives inside the product budget and ambiguous cases are safely diverted. For overnight work, a slower candidate may win by clearing more invoices without review.

Avoid a single weighted score. It conceals the failure that matters. Publish a small matrix instead: field accuracy by document stratum, unsafe accepts, review share, schema failures, p50 latency, and tail latency. Keep token estimates beside it, but never let price erase a quality violation. The tradeoff is explicit: a faster model earns the interactive lane only after it meets the quality gate, while a slower model can earn the batch lane by reducing manual review.

That is the gate.

## Roll Out with a Reversible Queue Split

Begin with a labeled shadow run. The new path may extract and validate, but it must not post accounting changes. Compare its decisions with the established result and have reviewers adjudicate disagreements, especially currency, credit-note status, and total selection.

Next, allow automatic acceptance only for a narrow stratum, such as clear single-currency invoices from approved suppliers, while every other document retains the review path. Increase the share by document class rather than by random global percentage. This makes rollback precise: disable one class without disturbing invoices whose evidence remains strong.

Finally, replay a fixed regression set before changing a model, prompt, label taxonomy, or routing rule. Monitor the acceptance and review distributions after rollout. A sudden speed improvement paired with more `unknown` labels is not a win; it is work displaced into a queue that somebody still has to clear.

The durable choice is the architecture, not the vendor name: strict labels, deterministic validation, confidence-gated escalation, separate interactive and batch lanes, and a corpus that reflects actual supplier messiness. With those controls in place, provider changes become measured routing decisions instead of migrations driven by anecdotes.

## Sources

References used for the comparison method and implementation context:

- OpenAI API documentation: https://platform.openai.com/docs/
- Anthropic API documentation: https://docs.anthropic.com/
- Google Gemini API documentation: https://ai.google.dev/gemini-api/docs
- Mistral AI API documentation: https://docs.mistral.ai/api/
- Groq API documentation: https://console.groq.com/docs/overview
- LiteLLM open-source LLM gateway: https://github.com/BerriAI/litellm
- MDN guide to Server-Sent Events: https://developer.mozilla.org/en-US/docs/Web/API/Server-sent_events/Using_server-sent_events
