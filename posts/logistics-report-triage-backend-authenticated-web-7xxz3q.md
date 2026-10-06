# Logistics Report Triage Backend: Authenticated Web App Chatbot Accounting

Route every model call through an authenticated server boundary, stream only presentation events to the browser, and commit usage to a tenant ledger after the provider reports it. **Short answer:** the best simple backend API is the one that keeps identity, moderation evidence, and cost attribution together while keeping the model adapter replaceable. For a logistics team classifying moderation reports before human review, that means choosing an API shape by its audit trail, not by how few lines its SDK demo needs.

The evidence is operational. Browser credentials are the wrong authority for model spend. A dropped stream does not prove that upstream generation stopped. Token estimates are useful for admission control but do not belong on an invoice. These boundaries matter more than the choice between Server-Sent Events (SSE) and a streamed `fetch()` response.

This architecture decision record assumes an authenticated web application where agents submit reports such as suspected shipment fraud, abusive driver messages, or prohibited-item claims. The model proposes a category and rationale; a human remains the reviewer. The primary selection axis is whether every unit of reported usage can be reconciled to the correct tenant without trusting browser-supplied tenant IDs.

## What should an authenticated web app chatbot backend own?

The first invariant is ownership: the server derives `tenant_id` and `actor_id` from the verified session. They are never accepted from the chat payload. OWASP's guidance is blunt here: authorization checks must occur on every request, and access control should be enforced in trusted server-side code.

The second invariant is evidence. Give each classification attempt an immutable request ID, record the model identifier and adapter name, and preserve a versioned prompt or policy reference. Store the final category separately from the human disposition. Otherwise a later reviewer cannot distinguish what the model proposed from what an operator approved.

The third invariant is accounting: reserve a tenant-specific budget before dispatch, then replace that reservation with provider-reported input and output usage when the call reaches a terminal state. Keep estimates labeled as estimates. **A client disconnect changes delivery state, not necessarily generation state.** This approach has a real trade-off: the server owns more state and must reconcile it. That is preferable here because attribution errors cross tenant and compliance boundaries, but it may be needless machinery for a synthetic-data prototype.

One more rule comes from OTP and messaging systems: “sent” and “received” are different events. The same distinction applies here. Picture a report accepted for tenant A, followed by one visible token and a closed laptop. The runtime may continue generation and report usage after the browser disappears; meanwhile the agent signs back in, retries, and creates another review candidate. If the application treats the socket close as cancellation, marks the first request failed, and charges only the second request, its review queue and runtime account now tell different stories. A durable request record resolves the ambiguity: the retry looks up the first request, delivery gets a new stream ID, and late usage attaches to the original tenant entry. Collapsing acceptance, generation, delivery, and review into one `success` boolean creates exactly this gap.

Four states are not one.

## Failure boundaries and the event contract

Use 3 identifiers with different jobs. A `request_id` makes retries idempotent. A `stream_id` identifies one delivery attempt. A trace ID connects authentication, classification, upstream generation, and ledger writes without becoming a business key. W3C Trace Context defines interoperable `traceparent` and `tracestate` headers, while OpenTelemetry defines conventions for recording generative AI operations; neither replaces the application ledger.

The browser should receive a deliberately small vocabulary of 4 event types: `accepted`, `delta`, `completed`, and `failed`. Do not expose raw provider chunks. Normalize them behind an adapter so provider-specific finish reasons, usage envelopes, or tool events cannot leak into the UI contract.

Small is enforceable.

Keep error semantics boring. Reject an unauthenticated request before opening the stream. Return a stable request ID after admission. Once response streaming has begun, report a terminal application event because the HTTP status can no longer be changed reliably. Persist the terminal state before, or atomically with, emitting `completed`; reconnecting clients can then query the authoritative result instead of guessing from the last visible token.

Backpressure deserves attention too. The Streams API exposes readable streams in browsers, and SSE has standardized event framing and automatic reconnection behavior. Neither protocol guarantees that an upstream model call is cancelled when a tab closes. Cancellation must be explicit, bounded by policy, and followed by reconciliation of whatever usage the runtime ultimately reports.

## Options compared by tenant visibility

The protocol is a secondary choice. What matters is where authentication, normalization, and the ledger live.

| Option | Authentication boundary | Tenant cost evidence | Recovery behavior | Appropriate use |
|---|---|---|---|---|
| Browser calls a model runtime directly | Browser or delegated token | Fragmented; browser metadata is untrusted | UI retry may duplicate work | Disposable prototypes with no sensitive reports or chargeback |
| Thin server proxy streams opaque chunks | Server authenticates, provider contract leaks through | Possible, but usage and completion handling vary by adapter | Disconnect handling is adapter-specific | One-runtime pilots where migration is explicitly out of scope |
| Server owns a normalized event stream and append-only usage ledger | Server session defines tenant and actor | Reported usage is tied to request, tenant, and terminal state | Idempotent lookup separates retry from replay | Multi-tenant moderation with review and audit requirements |
| Asynchronous job plus result polling | Server authenticates job creation and reads | Strong, because completion is detached from the connection | Natural replay and long-running recovery | Batch queues or classifications that do not need token-by-token display |

The third option fits an in-app chat experience while keeping per-tenant accounting visible. Its main limitation is operational: durable streaming needs terminal-state reconciliation and retention rules. The fourth option is often cleaner for pure classification. If users do not benefit from watching a rationale appear token by token, streaming adds delivery states, cancellation races, and partial-output policy questions for little operational gain.

That is the uncomfortable choice. “Chatbot” describes the interface, not necessarily the execution model.

## Critical path in Python

The following framework-neutral Python sketch shows the contract that an implementation should preserve. `SessionVerifier`, `RuntimeAdapter`, `Ledger`, and `EventSink` are application interfaces, not vendor SDK types. The ledger owns idempotency and the adapter returns authoritative usage when available.

```python
from dataclasses import dataclass
from typing import AsyncIterator, Protocol


@dataclass(frozen=True)
class Principal:
    tenant_id: str
    actor_id: str


@dataclass(frozen=True)
class Usage:
    input_tokens: int
    output_tokens: int


@dataclass(frozen=True)
class RuntimeEvent:
    kind: str
    text: str = ""
    usage: Usage | None = None


class RuntimeAdapter(Protocol):
    async def classify(
        self, *, request_id: str, report: str, policy_version: str
    ) -> AsyncIterator[RuntimeEvent]: ...


class Ledger(Protocol):
    async def begin(
        self, *, request_id: str, tenant_id: str, actor_id: str
    ) -> bool: ...

    async def complete(self, *, request_id: str, usage: Usage) -> None: ...

    async def fail(self, *, request_id: str, reason: str) -> None: ...


async def classify_report(
    *,
    principal: Principal,
    request_id: str,
    report: str,
    policy_version: str,
    runtime: RuntimeAdapter,
    ledger: Ledger,
) -> AsyncIterator[dict[str, object]]:
    created = await ledger.begin(
        request_id=request_id,
        tenant_id=principal.tenant_id,
        actor_id=principal.actor_id,
    )
    if not created:
        yield {"type": "failed", "request_id": request_id, "code": "duplicate"}
        return

    yield {"type": "accepted", "request_id": request_id}
    try:
        terminal_usage: Usage | None = None
        async for event in runtime.classify(
            request_id=request_id,
            report=report,
            policy_version=policy_version,
        ):
            if event.kind == "delta":
                yield {"type": "delta", "request_id": request_id, "text": event.text}
            elif event.kind == "usage":
                terminal_usage = event.usage

        if terminal_usage is None:
            await ledger.fail(request_id=request_id, reason="usage_unavailable")
            yield {
                "type": "failed",
                "request_id": request_id,
                "code": "usage_unavailable",
            }
            return

        await ledger.complete(request_id=request_id, usage=terminal_usage)
        yield {"type": "completed", "request_id": request_id}
    except Exception:
        await ledger.fail(request_id=request_id, reason="runtime_error")
        yield {"type": "failed", "request_id": request_id, "code": "runtime_error"}
```

Production code needs a transaction or an outbox around terminal ledger state and result publication. It also needs a strict report-size limit, tenant-level concurrency control, and a redaction policy for traces. Do not put report text, rationale text, session tokens, or personal data into span attributes merely because tracing makes that convenient. OpenTelemetry's generative AI conventions explicitly discuss opt-in capture for sensitive input and output content.

The exception boundary above intentionally emits a generic code. Detailed upstream errors belong in access-controlled logs, correlated by request ID. This is the same discipline that keeps an SMS provider response from leaking into an end-user error: operators need detail; clients need stable semantics.

## Reconciliation is the selection test

Before choosing a runtime or API library, run a small failure matrix against the adapter. Use 2 tenants, 2 actors per tenant, and at least 6 cases: normal completion, browser disconnect after the first delta, repeated request ID, runtime timeout, missing usage, and a reconnect after terminal persistence. Verify ledger rows and review records, not just rendered text.

Five columns are enough to expose most design mistakes: `request_id`, `tenant_id`, `usage_source`, `terminal_state`, and `recorded_at`. Add model and policy versions for auditability. Monetary conversion should happen in a separate rate table with effective dates; raw token or unit counts are more durable than a precomputed currency amount. Pricing changes. Evidence should not.

For each tenant, reconcile three totals over the same window: admitted requests, terminal requests, and provider-reported usage. They answer different questions. A mismatch between admitted and terminal requests indicates stranded work. Usage without a terminal request indicates a persistence or correlation defect. Terminal records with estimated usage should stay visibly provisional rather than quietly entering chargeback.

This is also where embeddings fit, if the classifier retrieves policy examples. Embeddings represent text as vectors and support similarity-based retrieval, but retrieval usage and generation usage are separate ledger entries. Record the policy corpus version and retrieval operation independently. Otherwise a seemingly cheap classifier can hide a second workload whose tenant attribution is impossible to reconstruct.

## Rejected option and its valid use case

The rejected design is direct browser-to-runtime streaming with a short-lived credential. It removes a server hop, and that can be valid for a single-tenant prototype using synthetic data, no human-review audit trail, and no internal chargeback. It is also useful when the browser is the intended trust boundary and the runtime can enforce the complete authorization policy itself.

It does not fit this logistics moderation system. Tenant identity would cross a boundary controlled by the client, provider events would become part of the UI contract, and disconnect reconciliation would have no durable coordinator. Adding a backend later would change authentication, event shapes, retry rules, and accounting at once.

Choose the smallest server contract that preserves the invariants: verified identity in, normalized events out, and durable usage tied to one tenant and request. **The runtime remains replaceable because the audit model does not.**

## References

- OWASP Authorization Cheat Sheet: https://cheatsheetseries.owasp.org/cheatsheets/Authorization_Cheat_Sheet.html
- W3C Trace Context: https://www.w3.org/TR/trace-context/
- OpenTelemetry semantic conventions for generative AI systems: https://opentelemetry.io/docs/specs/semconv/gen-ai/
- MDN, Using server-sent events: https://developer.mozilla.org/en-US/docs/Web/API/Server-sent_events/Using_server-sent_events
- MDN, Streams API: https://developer.mozilla.org/en-US/docs/Web/API/Streams_API
- OpenAI Embeddings guide: https://platform.openai.com/docs/guides/embeddings
- Prompt Engineering Guide: https://www.promptingguide.ai
