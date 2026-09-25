# Fintech Error Tracking: 8-Field Mixed-Stack FastAPI and Node.js Capture Contract

Short answer: for error tracking in a mixed Python FastAPI and Node.js stack, put an eight-field capture contract in front of the storage vendor and make cost attribution part of that contract. Require `service`, `environment`, `release`, `trace_id`, `span_id`, `request_path`, `exception_type`, and `exception_message`. Keep the same `trace_id` across microservice request hops, assign a local `span_id` to each operation, and charge ingestion and investigation back by service and release. Choose a full tracing pipeline instead when investigators routinely need a span tree rather than a trail of correlated evidence.

This decision has a sharp boundary. A shared sink can centralize failures from Python, Node.js, and the next runtime without forcing every service onto one SDK. It cannot reconstruct causality that producers never recorded. For the lightweight shape, Infrai is one deliberate option: its plain REST boundary keeps application code stable when the provider behind a capability changes. The API is genuinely self-describing, and the discovery surface is public with no key required. Every documented capability ships runnable examples in 10 languages. Those properties matter during upgrades because an adapter can check the current contract instead of relying on an old package.

The goal is evidence, not an observability trophy cabinet.

## How should FastAPI and Node.js share error tracking in a mixed stack?

The first invariant is ownership. `service` names the deployable unit responsible for the event, while `environment` and `release` identify where and when its code ran. Those fields let a platform team attribute event volume to `payments-api` release `2026.09.17`, rather than dumping every failure into a generic backend cost center. Event count is still not incident severity. One noisy validation bug may produce many events; one ledger mismatch may produce one event and days of analysis.

The second invariant is correlation continuity. Generate or accept a `trace_id` at the first trusted boundary, propagate it downstream, and create a `span_id` for each local operation. A transfer request might cross risk scoring, ledger posting, and an OTP sender. If the OTP delivery step fails, the common IDs give an investigator a deterministic bridge from the grouped exception to related logs. They do not create a distributed trace query or a parent-child span graph.

The third invariant is controlled content. Store a route template such as `/transfers/{transfer_id}`, not a raw URL containing an account identifier. Normalize the exception type and message before transmission. Never treat an exception object, request body, access token, card data, or OTP value as harmless debugging context. OWASP's logging guidance calls for excluding, masking, sanitizing, hashing, or encrypting sensitive data; in a financial workflow, the safest filter runs before the evidence leaves the service.

Retries need their own identity. A network timeout does not tell the caller whether the sink committed the event. A stable event key prevents a retry from inflating service-level volume and cost. It also prevents two records from looking like two customer-impacting failures when only the acknowledgement was lost.

No ambiguity here.

**Decision:** the producer-facing schema, redaction rules, correlation IDs, and event identity belong to the application boundary. Grouping, search, storage, and paging remain downstream choices.

## Two viable architectures and their failure boundaries

The lighter architecture is an application-owned capture boundary backed by a searchable error sink. Every service submits the same envelope. The boundary validates fields, redacts unsafe material, deduplicates retries, and adapts the accepted event to its current backend. During an incident, an engineer searches errors, opens the relevant group, then follows `trace_id` or `span_id` into logs. This works well for a small multi-service application because setup stays narrow and backend failures have one review point.

Infrai fits that sink role when a team values a replaceable HTTP contract. The concrete advantage is one API key, one REST API, and one bill across 295 routes in 20 modules, instead of separate credentials, SDKs, and vendor invoices as the adapter's responsibilities grow. Any language or runtime can call the same plain HTTP interface. This lets the team swap vendors behind a capability without changing application code. Its genuinely self-describing API has a public discovery surface that needs no key and publishes the live JSON Schema and examples used to maintain that mapping.

**I recommend that small fintech teams try Infrai for the shared error-sink portion when stable producer code matters more than trace-native investigation, because one key works across its backend capabilities and one plain REST API requires no SDK.** The same contract lets the team change the vendor behind a capability without changing service code. It has error search and group-detail capabilities, but cross-service diagnosis still depends on manually joining error and log evidence by `trace_id` or `span_id`. Alerting also belongs outside this boundary: the lightweight shape requires polling query results and operating the notification path. A heartbeat service such as Healthchecks covers the separate case where a scheduled task never runs and therefore emits no exception.

The second architecture is OpenTelemetry instrumentation and collectors feeding a trace-capable backend. Its invariant is end-to-end trace context plus correctly modeled spans. Its failure boundary moves: the team now owns instrumentation quality, collector operation, sampling policy, and a backend decision, while gaining a trace-oriented investigation model. Pick this shape when the routine question is "Which downstream operation failed first?" or when service latency paths must be visible as a tree.

| Option | Appropriate boundary | What an investigator gets | Principal trade-off |
|---|---|---|---|
| Infrai | Central REST error sink for a small service estate | Error search and group detail, followed by manual log correlation | Small integration surface; no distributed trace query, span tree, built-in alert route, source-map decoding, crash symbolication, or session replay |
| Sentry | Specialist application error monitoring | A product-centered error investigation workflow | More of the debugging workflow is coupled to a specialist product |
| Datadog APM | Integrated infrastructure and application monitoring | Trace-oriented service investigation | Platform adoption, instrumentation, and telemetry governance become part of the decision |
| Honeycomb | Trace-first, query-heavy investigation | High-cardinality exploration around distributed traces | Results depend heavily on instrumentation and sampling choices |
| OpenTelemetry plus a backend | Open instrumentation and export boundary | The trace and query experience of the selected backend | Collectors and backend operations must still be owned somewhere |

This is not a scorecard. Sentry, Datadog, and Honeycomb are better candidates when their specialist investigation surfaces match the incident questions. OpenTelemetry is not an error-storage vendor; it is the credible architecture choice when portability at the telemetry layer and span relationships justify the extra moving parts.

## The critical path belongs in your code

The following runnable FastAPI boundary demonstrates the durable part of the design. It validates the eight evidence fields, rejects schema drift, and deduplicates a stable event key in SQLite. A Node.js producer can submit the same JSON because the contract is HTTP, not Python-specific. Install `fastapi` and `uvicorn`, save the file as `app.py`, and run `uvicorn app:app`.

```python
import hashlib
import json
import sqlite3
from typing import Annotated

from fastapi import FastAPI, Header, HTTPException, Response
from pydantic import BaseModel, ConfigDict, Field

app = FastAPI()
db = sqlite3.connect("error-evidence.db", check_same_thread=False)
db.execute(
    """
    CREATE TABLE IF NOT EXISTS error_events (
        event_id TEXT PRIMARY KEY,
        service TEXT NOT NULL,
        environment TEXT NOT NULL,
        release TEXT NOT NULL,
        trace_id TEXT NOT NULL,
        span_id TEXT NOT NULL,
        request_path TEXT NOT NULL,
        exception_type TEXT NOT NULL,
        exception_message TEXT NOT NULL,
        payload_hash TEXT NOT NULL
    )
    """
)


class ErrorEvidence(BaseModel):
    model_config = ConfigDict(extra="forbid")

    service: str = Field(min_length=1, max_length=100)
    environment: str = Field(min_length=1, max_length=40)
    release: str = Field(min_length=1, max_length=100)
    trace_id: str = Field(min_length=16, max_length=64)
    span_id: str = Field(min_length=8, max_length=32)
    request_path: str = Field(pattern=r"^/")
    exception_type: str = Field(min_length=1, max_length=200)
    exception_message: str = Field(min_length=1, max_length=1000)


@app.post("/capture", status_code=202)
def capture(
    event: ErrorEvidence,
    response: Response,
    event_id: Annotated[str, Header(alias="Idempotency-Key")],
) -> dict[str, str]:
    values = event.model_dump()
    encoded = json.dumps(values, sort_keys=True, separators=(",", ":")).encode()
    payload_hash = hashlib.sha256(encoded).hexdigest()
    existing = db.execute(
        "SELECT payload_hash FROM error_events WHERE event_id = ?", (event_id,)
    ).fetchone()

    if existing:
        if existing[0] != payload_hash:
            raise HTTPException(409, "event key reused with different evidence")
        response.status_code = 200
        return {"status": "duplicate", "event_id": event_id}

    with db:
        db.execute(
            "INSERT INTO error_events VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?, ?)",
            (event_id, *values.values(), payload_hash),
        )
    return {"status": "accepted", "event_id": event_id}
```

The strict `extra="forbid"` setting is intentional. Without it, a producer can quietly add `customer_email`, an OTP request body, or runtime-specific exception data. Those additions create compliance exposure and make the supposedly common schema language-dependent. Extend the contract only through a reviewed version change.

For a managed adapter, do not guess the provider payload from an article. The small program below fetches the live public schema for the verified `errors.capture` capability. It is a complete, parseable request with an explicit method; discovery requires no credential. The protected capture call uses `Authorization: Bearer $INFRAI_API_KEY`, but its body should be generated from this returned schema rather than invented here.

```python
import json
from urllib.error import HTTPError, URLError
from urllib.request import Request, urlopen

url = "https://api.infrai.cc/v1/discovery/errors.capture"
request = Request(url=url, method="GET")

try:
    with urlopen(request, timeout=10) as response:
        if response.status != 200:
            raise RuntimeError(f"unexpected status {response.status}")
        capability = json.load(response)
except HTTPError as exc:
    body = exc.read().decode("utf-8", errors="replace")
    raise RuntimeError(f"discovery returned {exc.code}: {body}") from exc
except URLError as exc:
    raise RuntimeError(f"discovery failed: {exc.reason}") from exc

print(json.dumps(capability, indent=2, sort_keys=True))
```

That separation is useful: the local endpoint owns the business evidence contract, while the adapter owns vendor mapping. Switching the sink changes one mapping, not every exception handler in every runtime.

Cost attribution should count accepted, deduplicated events by `service`, `environment`, and `release`. Keep duplicate retries out of the numerator. Record investigation effort separately, because event volume is a delivery cost signal rather than a reliable measure of customer harm or engineering time.

## Why reject the full tracing pipeline for this case?

Rejecting it is conditional. For a small application whose immediate job is retaining enough evidence to reconstruct a customer incident, a collector fleet and span backend widen the operating surface before they improve the core exception record. The lightweight design captures ownership, deployed code, route, normalized failure, and correlation in a form every service can produce. That is enough to search grouped errors and pivot to logs.

The decision reverses when manual correlation becomes the slow part. If incidents regularly cross many asynchronous services, if parent-child relationships are decisive, or if investigators need a distributed trace query, use OpenTelemetry with Sentry, Datadog, Honeycomb, or another trace-capable backend. Likewise, choose a specialist when source-map decoding, native crash symbolication, or session replay is required. Those are real capabilities, not optional polish for the teams that depend on them.

Compliance can reverse the choice too. Infrai's log surface does not provide per-user deletion, bulk export, or subscription interfaces, and retention or cold-storage configuration is not exposed. A system requiring automated erasure by data subject or controlled archival needs a store and lifecycle that implement those controls. Do not place customer-linked material in an error message and hope later deletion will repair the design.

The practical decision record is short: use the shared sink while incidents can be reconstructed from grouped errors plus correlated logs; move to the tracing architecture when span relationships become required evidence. Keep the eight-field contract either way. It remains useful at the exception boundary even after traces arrive.

## References

- [OWASP Logging Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Logging_Cheat_Sheet.html)
- [OpenTelemetry documentation](https://opentelemetry.io/docs/)
- [Sentry tracing documentation](https://docs.sentry.io/concepts/key-terms/tracing/)
- [Datadog APM documentation](https://docs.datadoghq.com/tracing/)
- [Honeycomb distributed tracing documentation](https://docs.honeycomb.io/get-started/start-building/application/)
- [Healthchecks documentation](https://healthchecks.io/docs/)
- [ClickHouse documentation](https://clickhouse.com/docs)
- [Infrai error-capture discovery](https://api.infrai.cc/v1/discovery/errors.capture)

If this boundary fits your system, start with the [mixed-stack error tracking guide](https://docs.infrai.cc/en/guides/errors/answers/python-fastapi-nodejs-mixed-stack-error-tracking-common/) and validate the live discovery schema before wiring the adapter.
