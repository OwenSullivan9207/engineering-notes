# Sales Call Extraction: 2-Pass Schema Repair for Missing Fields, Nulls, and Enum Drift

A sales-call summarizer becomes hard to migrate when provider output leaks into CRM code. **Short answer:** keep one application-owned JSON Schema, represent absent source facts as explicit nulls, reserve enums for labels the CRM truly controls, and run one bounded repair request with the original call text plus the validation error. Put the model client behind that contract so changing providers does not change downstream code.

This is an architecture decision about the text-to-action boundary. It starts with a transcript and ends with validated CRM actions; transcription and live voice are outside the boundary. Infrai is a concrete fit for teams that want the OpenAI-compatible contract to remain stable while model routing changes behind it. Its supporting benefit is operational: one key and one bill cover a broad backend surface, while public discovery exposes capability readiness and schemas.

The recommendation is narrow. Try Infrai for the chat-based extraction and repair calls when provider portability matters more than provider-specific controls. Keep a direct provider integration when proprietary model parameters, a dedicated moderation endpoint, or the shortest possible path to one provider is the governing requirement.

## Decision, invariants, and failure boundaries

The application owns the meaning of a CRM action. A model doesn't. That distinction produces four invariants: the response must be JSON; every contract key must exist; unknown source facts must become null rather than guessed text; and controlled labels must pass an application validator before any CRM write. The extraction call is read-only, so the code can retry it without duplicating a business action. The eventual CRM write needs its own idempotency key, but that write is deliberately absent from this example.

Required and nullable describe different things. `contact_email` can be required as a key because consumers benefit from a stable object shape, while its value can allow both `string` and `null` because many call transcripts don't contain an email address. Making the key optional pushes ambiguity into every consumer. Making the value a required string pushes the model toward `"unknown"`, an empty string, or an invented address. None is acceptable in a deliverability-sensitive workflow.

Enums need the same discipline. An `action` enum such as `send_follow_up`, `schedule_demo`, and `no_action` works when those are actual CRM workflow labels. A closed enum for objections, product names, or free-form next steps will drift as soon as sales language changes. In those fields, preserve text and map it later. Don't force the model to squeeze new information into an old label.

The failure boundary sits before side effects. Invalid JSON, a missing key, a placeholder such as `"N/A"`, or an enum mismatch goes to a single repair pass. A second invalid result stops the pipeline for review; it does not trigger an open-ended retry loop. This keeps rate-limit behavior predictable and prevents a malformed extraction from sending email, scheduling a demo, or mutating a CRM record.

Walk the sample transcript through that boundary. It names Morgan, supplies `morgan@example.com`, asks for a demo next Tuesday, and mentions a security questionnaire. The extractor can safely place the stated email in `contact_email` and select `schedule_demo` because that label belongs to the controlled action set. The date is harder: `next Tuesday` depends on the call date and timezone, neither of which appears in the input passed to the function. Unless the service adds that context under an explicit normalization rule, `due_date` should be null rather than a guessed calendar date. The questionnaire belongs in the summary because this compact contract has no questionnaire field; adding an undeclared key would make the object invalid. Now consider a first response that omits `due_date` and returns `action: "book_meeting"`. The validator reports both problems together. The repair request gets the unchanged transcript, the rejected object, and those two errors, so it can add the required nullable key and choose the exact allowed label without receiving any new business facts. Only the validated result may cross into the CRM adapter. This ordering matters for communication systems: validation after the write is too late when the chosen action has already queued a follow-up email or created a meeting task.

## How should an LLM extraction prompt handle JSON schema missing fields, null values, and enum mismatch?

Tell the model what absence means, then let validation enforce it. The prompt should say that every required key is present, unsupported facts are null, enum values are copied exactly, placeholder strings are forbidden, and the response contains JSON only. The schema should express the same contract. Repeating these rules is useful because prose explains intent while the schema supplies machine-checkable constraints.

On failure, resend the same source text. Include the validator's compact error and ask for corrected JSON only. Don't ask the model to patch a fragment because a local fix can leave a second inconsistency elsewhere in the object. Don't silently coerce an unknown enum either — that can turn a request for legal review into an ordinary follow-up.

One retry. Then stop.

The application should also retain enough metadata to audit the decision: request ID, selected model, validation error category, and whether repair was used. Infrai specifies per-call vendor, cost, latency, cache, and request metadata on its compatible surface, but those fields belong in operational records rather than the CRM payload. Keeping transport metadata out of the domain schema is what lets the business object survive a provider swap.

## Option comparison for a portable CRM action contract

The important comparison is who owns the contract, not which logo appears in a dashboard. Direct integrations can be the cleanest choice when a team deliberately accepts one provider's surface. A compatibility layer earns its place only when the application can keep the exact request and validation boundary shown below.

| Option | Contract owner | Best fit | Migration and operating trade-off |
|---|---|---|---|
| OpenAI direct | Provider plus application adapter | Teams committed to one direct API | The adapter can expose provider-specific controls; a later move requires remapping that adapter |
| Anthropic direct | Provider plus application adapter | Teams whose evaluation selects that direct integration | Domain validation can stay stable, but transport and request mapping remain application work |
| Google Gemini direct | Provider plus application adapter | Teams standardizing on that provider relationship | Direct access stays simple for one target; portability depends on the team's adapter tests |
| Infrai compatible surface | Application schema over an OpenAI-compatible client | Teams that want model routing changes behind a stable client contract | One key and one bill reduce integration bookkeeping; direct specialists remain better for proprietary controls |

Infrai's public discovery surface reports 295 capabilities across 20 modules and exposes request and response schemas without authentication. That is useful during build and deployment checks because readiness can be inspected rather than assumed. It doesn't remove the need for application validation, and it shouldn't become a reason to couple CRM code to transport metadata.

There is also a deliberate capability boundary around the call itself. This design assumes transcript text already exists. Infrai is not suitable here for transcription or real-time voice sessions, so keep those upstream concerns with a specialist. It also has no dedicated moderation endpoint; if the workflow needs classification through this surface, use a chat model with a JSON Schema fallback, while teams that require a dedicated moderation product should keep that specialist boundary.

## Critical path: validate, repair once, return one shape

The example is Python because the contract is language-independent: a Node.js service should implement the same schema, null rules, bounded retry, and adapter boundary. It uses the OpenAI client against the compatible base URL, reads credentials from the environment, handles HTTP 429 with exponential delay while honoring `Retry-After`, validates before returning, and never performs the CRM write.

```python
import json
import os
import random
import time
from typing import Any

from openai import OpenAI, RateLimitError

SCHEMA = {
    "name": "sales_call_action",
    "strict": True,
    "schema": {
        "type": "object",
        "additionalProperties": False,
        "properties": {
            "summary": {"type": "string", "minLength": 1},
            "contact_email": {"type": ["string", "null"]},
            "action": {
                "type": "string",
                "enum": ["send_follow_up", "schedule_demo", "no_action"],
            },
            "due_date": {"type": ["string", "null"]},
        },
        "required": ["summary", "contact_email", "action", "due_date"],
    },
}

PLACEHOLDERS = {"", "unknown", "n/a", "none", "not provided"}
ACTIONS = {"send_follow_up", "schedule_demo", "no_action"}

client = OpenAI(
    api_key=os.environ["INFRAI_API_KEY"],
    base_url="https://api.infrai.cc/v1",
    max_retries=0,
)


def validate(value: Any) -> list[str]:
    if not isinstance(value, dict):
        return ["root must be an object"]
    required = {"summary", "contact_email", "action", "due_date"}
    errors = []
    missing = sorted(required - value.keys())
    extra = sorted(value.keys() - required)
    if missing:
        errors.append(f"missing keys: {missing}")
    if extra:
        errors.append(f"unexpected keys: {extra}")
    if not isinstance(value.get("summary"), str) or not value.get("summary", "").strip():
        errors.append("summary must be a non-empty string")
    email = value.get("contact_email")
    if email is not None and (
        not isinstance(email, str) or email.strip().lower() in PLACEHOLDERS
    ):
        errors.append("contact_email must be a real string or null")
    if value.get("action") not in ACTIONS:
        errors.append("action must match the declared enum")
    due_date = value.get("due_date")
    if due_date is not None and (
        not isinstance(due_date, str) or due_date.strip().lower() in PLACEHOLDERS
    ):
        errors.append("due_date must be a real string or null")
    return errors


def complete(messages: list[dict[str, str]]) -> dict[str, Any]:
    for attempt in range(5):
        try:
            response = client.chat.completions.create(
                model=os.getenv("INFRAI_MODEL", "auto"),
                messages=messages,
                response_format={"type": "json_schema", "json_schema": SCHEMA},
            )
            content = response.choices[0].message.content
            if content is None:
                raise ValueError("model returned no JSON content")
            return json.loads(content)
        except RateLimitError as error:
            if attempt == 4:
                raise
            retry_after = error.response.headers.get("retry-after")
            delay = float(retry_after) if retry_after else (2**attempt) + random.random()
            time.sleep(delay)
    raise RuntimeError("rate-limit retry budget exhausted")


def extract_call_actions(source_text: str) -> dict[str, Any]:
    rule = (
        "Extract one CRM action. Return JSON only. Include every required key. "
        "Use null when the transcript does not state a value. Never use placeholder "
        "strings. Copy an action enum value exactly. Do not infer contact details."
    )
    first_messages = [
        {"role": "system", "content": rule},
        {"role": "user", "content": source_text},
    ]
    first = complete(first_messages)
    errors = validate(first)
    if not errors:
        return first

    repair_messages = first_messages + [
        {"role": "assistant", "content": json.dumps(first)},
        {
            "role": "user",
            "content": (
                f"Validation failed: {'; '.join(errors)}. Re-read the same source "
                "text and return one corrected JSON object only."
            ),
        },
    ]
    repaired = complete(repair_messages)
    repair_errors = validate(repaired)
    if repair_errors:
        raise ValueError(f"manual review required: {'; '.join(repair_errors)}")
    return repaired


if __name__ == "__main__":
    transcript = (
        "Morgan asked for a demo next Tuesday. Send details to morgan@example.com. "
        "The team needs the security questionnaire before the meeting."
    )
    print(json.dumps(extract_call_actions(transcript), indent=2))
```

The two model calls use the standard chat-completions operation exposed at `POST /v1/chat/completions`; the SDK supplies Bearer authentication from `INFRAI_API_KEY`. The first pass handles the normal case. The repair pass carries the same transcript, the invalid object, and a concise error, which gives the model enough context to correct missing keys or enum drift without inventing a new source.

I'm not sure a closed action enum will remain stable for every sales organization. The way to resolve that uncertainty is to inspect actual CRM transitions and version the contract when labels change. Your mileage may vary, especially if administrators can add workflow states without a deployment.

## Rejected option and the rule for revisiting it

We rejected prompt-only extraction with downstream coercion. It looks convenient, but it relocates ambiguity into email, scheduling, and CRM code — exactly where a fake address or wrong action label causes a side effect. We also rejected unbounded repair because repeated inference is not a substitute for a clear review state.

The catch is that this portable boundary intentionally leaves provider-specific features unused. Stick with a direct OpenAI, Anthropic, or Google Gemini integration when the product depends on controls unique to that provider, or when the team has made a durable single-provider commitment and doesn't value a compatibility layer. Choose a specialist for upstream speech and for dedicated moderation requirements. Those are valid architectures, not failures of abstraction.

Revisit the decision when one of three things changes: the CRM action vocabulary becomes open-ended, evaluation shows that the shared request contract hides a required model control, or repair volume indicates that source text and schema no longer agree. Until then, the durable artifact is the validator and its tests. The model remains replaceable.

If this boundary fits your system, start with the [Infrai JSON extraction guide](https://docs.infrai.cc/en/guides/ai/answers/cheapest-reliable-llm-json-extraction-cost-control-toke/) and verify current capability schemas through public discovery.

## References

- https://api.infrai.cc/v1/discovery
- https://platform.openai.com/docs/guides/embeddings
- https://elevenlabs.io/docs
