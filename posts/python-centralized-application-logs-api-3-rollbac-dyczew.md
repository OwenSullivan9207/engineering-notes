# Python Centralized Application Logs API — 3 Rollback-Safe Ingestion and Search Signals

Use an OpenTelemetry-compatible log ingestion API, keep search behind a separate read API, and alert on three explicit signals: the scheduled attempt, its terminal outcome, and the number of accepted results. For a fintech import, this split is easier to operate than treating a dashboard search as the monitoring system. More important, it is rollback-safe: an older worker can resume emitting the stable event contract while a newer detector or dashboard is reverted independently.

That is the decision. The deciding constraint is not how quickly a team can draw its first chart. It is whether a deployment rollback can hide a stopped import, double-count a retry, or make an old event unreadable. Logging must preserve enough evidence to answer those questions after the process that emitted it is gone.

This note records an architecture for a startup backend whose scheduled imports should continually produce results. The proposed boundary uses structured log records for durable evidence, a small stateful detector for liveness, and search for investigation. Alerts do not depend on someone running the right query.

## Which API should a startup use for centralized application logs ingestion?

Four invariants define the design.

1. Every scheduled opportunity has a stable `schedule_id`, even when no worker starts.
2. Every execution has a unique `run_id`, while retries retain the same `schedule_id` and increment `attempt`.
3. A run emits one terminal outcome: `succeeded`, `failed`, or `cancelled`. Success includes `accepted_count`, including zero.
4. The alerting clock is based on an expected deadline stored outside the worker release. Rolling back application code must not roll back knowledge of a missed schedule.

The third invariant matters in financial data flows. “The job ran” and “the job produced accepted records” are different claims. A syntactically valid upstream response can contain zero rows; a row can also be rejected by validation. Collapsing those cases into one success message makes the quietest failure look healthy.

Silence is data.

I use the same mental model for OTP delivery: acceptance, delivery, and user-visible completion are separate states. A provider accepting a message is not proof that a handset received it. Here, a scheduler dispatching work is not proof that an import committed results. The uncomfortable gap is where the useful alert belongs.

The event schema is deliberately boring. Use UTC timestamps, a severity, a machine-stable event name, identifiers, outcome fields, and a schema version. The OpenTelemetry Logs Data Model defines timestamps, observed timestamps, severity, body, attributes, resource context, and trace context; OTLP defines transport for telemetry data. Those standards make a reasonable ingestion boundary without dictating the storage engine or dashboard.

Do not put account numbers, access tokens, raw imported rows, or one-time codes into attributes. OWASP's logging guidance calls out secrets and sensitive personal data as values that usually should not be recorded directly. For investigation, retain opaque tenant and source identifiers whose mapping is access-controlled elsewhere.

## Failure boundaries and the 3 signals

The detector should consume the same immutable evidence as search, but it owns a small amount of derived state. For each schedule it tracks the most recent expected deadline, the latest attempt, the terminal outcome, and accepted result count. It then evaluates three signals.

**Missing attempt:** the schedule deadline passed without an `import.started` event. This catches a disabled scheduler, a queue publication failure, or a worker fleet that never received work. A worker-only heartbeat cannot see all three.

**Missing terminal outcome:** an attempt started but did not finish within its declared execution window. This catches a crashed or stalled worker. The timeout is configuration attached to the schedule, not a magic constant embedded in a query.

**No accepted results:** a terminal success repeatedly reports `accepted_count == 0`, according to a policy chosen for that feed. One empty run might be legitimate; five may be alarming for a daily settlement feed and normal for a rarely used account. The detector therefore needs an explicit consecutive-run or elapsed-time policy. It must not infer business cadence from whichever logs happen to remain searchable.

These are event-time decisions with processing-time safeguards. The record carries when the application says the event occurred, while the collector records when it observed the event. OpenTelemetry distinguishes `Timestamp` from `ObservedTimestamp`, which helps expose delayed delivery. The detector should allow a bounded lateness window and make that window visible in policy. During a collector backlog, it can report “telemetry delayed” separately from “import missing” rather than issuing a confident but false diagnosis.

The failure boundaries are equally important. The worker owns import execution and emitting run evidence. The telemetry pipeline owns accepting and durably buffering records. The detector owns schedule-state evaluation. The search service owns retrieval, retention, and access control. A dashboard only renders their outputs.

If the dashboard is unavailable, detection continues. If search indexing lags, ingestion acknowledgements and detector checkpoints remain distinct. If an application rollback emits schema version 1 after version 2 has appeared, consumers still accept version 1 until its documented retirement. Short and dull wins.

That compatibility window has a cost.

## Compare the ingestion choices

There are several plausible APIs, but they protect different failure boundaries.

| Option | What crosses the boundary | Rollback behavior | Main blind spot | Appropriate use |
|---|---|---|---|---|
| Structured logs over OTLP | Versioned log records and resource attributes | Old and new workers can coexist if consumers use additive schema evolution | A worker that never starts emits nothing | Default evidence path, paired with schedule state |
| Direct domain-event endpoint | Import-specific commands such as run started or completed | Strong contract, but application and detector releases are more tightly coordinated | Endpoint failure can enter the job's critical path | Regulated workflows needing a dedicated audit contract |
| Metrics-only push | Counters and gauges | Small surface, but label and semantic changes can split a time series | Individual attempts and rejection reasons are hard to reconstruct | Aggregate service health with logs retained elsewhere |
| Periodic database polling | Current run rows | A detector can be reverted independently of workers | Missing dispatches require a separate schedule source; overwritten state loses history | A small system whose run ledger is already authoritative |

The first option is the ADR choice, with one qualification: schedule expectations cannot live only in emitted logs. Store the schedule definition and its next expected deadline in an authoritative control table or scheduler state. The detector joins that expectation to ingested evidence. This closes the “nothing ran, therefore nothing logged” hole.

The trade-off is extra operational state. Teams must back up the detector checkpoint, reconcile it after replay, and monitor the telemetry path itself. This pattern is not suitable when the database run ledger already provides immutable history and the job is noncritical; polling that ledger can be easier to reason about. It is also a poor fit for a hard audit requirement that demands transactional coupling between a business commit and its evidence. In that case, an outbox-backed domain-event contract is the stronger boundary, even though it adds application schema and migration work.

Keep ingestion and search asymmetric. Ingestion is a narrow write contract optimized for validation, batching, backpressure, and durable acknowledgement. Search is a read contract optimized for bounded time ranges, tenant authorization, pagination, and selected fields. Reusing a search endpoint as a liveness probe couples alert correctness to index freshness and query availability.

For an early-stage system, require a time range and tenant scope on every search. Cap page size and use an opaque cursor. Those are design constraints, not claims about a particular product. They protect a shared backend from the dashboard query that accidentally asks to scan all history, and they make authorization review less ambiguous.

## Critical path in Python

The application should emit after each state transition, using an additive schema. The example below uses only the Python standard library and writes newline-delimited JSON to standard output, where a collector can batch and export it. In production, the process supervisor and collector must preserve stdout across normal restarts; the import transaction must not depend on a remote search service being reachable.

```python
import json
import logging
import sys
from datetime import datetime, timezone
from typing import Any
from uuid import uuid4


logger = logging.getLogger("scheduled_import")
logger.setLevel(logging.INFO)
logger.addHandler(logging.StreamHandler(sys.stdout))


def emit(event_name: str, **attributes: Any) -> None:
    record = {
        "timestamp": datetime.now(timezone.utc).isoformat(),
        "severity": "INFO",
        "event_name": event_name,
        "schema_version": 1,
        **attributes,
    }
    logger.info(json.dumps(record, separators=(",", ":"), sort_keys=True))


def run_import(schedule_id: str, tenant_id: str, attempt: int) -> None:
    run_id = str(uuid4())
    common = {
        "schedule_id": schedule_id,
        "tenant_id": tenant_id,
        "run_id": run_id,
        "attempt": attempt,
    }
    emit("import.started", **common)

    try:
        accepted_count = import_and_commit_rows(tenant_id)
    except Exception as exc:
        emit(
            "import.finished",
            **common,
            outcome="failed",
            error_type=type(exc).__name__,
        )
        raise

    emit(
        "import.finished",
        **common,
        outcome="succeeded",
        accepted_count=accepted_count,
    )
```

`import_and_commit_rows` is intentionally outside the example. Its return value must describe committed, accepted rows, not fetched rows. Emitting success before the database commit would create evidence for a result that a rollback can erase.

The exception records only its type. Dumping exception text without classification risks leaking upstream payloads or credentials, a particularly bad bargain in a fintech log store. A controlled error code can be added later. Add fields; do not silently change their meaning.

Delivery is normally at least once across process, collector, and storage failures, so consumers should deduplicate by a stable event identity or tolerate duplicate state transitions. The sample has stable run identity but no event identity because the exact retry boundary belongs to the surrounding transport. Whichever layer retries a record must preserve the same event ID; generating a new ID on every send defeats deduplication.

Test the contract with old and new producers at the same time. A release check should replay version 1 fixtures into the new detector, version 2 fixtures into the old-compatible search projection, duplicates in both orders, a terminal event arriving before its start event, and events beyond the lateness window. Also test absence: advance a fake clock past a scheduled deadline without inserting any event. That last test finds the class of bug that log-only happy-path fixtures miss.

Deployment follows the compatibility order: expand consumers, deploy producers, observe both schema versions, and only then contract consumers after the rollback window closes. Alert-policy changes deserve the same care. Version the policy, record which version opened an alert, and let a rollback continue evaluating already-open alerts under a defined rule rather than silently resetting them.

## The rejected shortcut, and when it is valid

The rejected design is a dashboard that periodically searches for the latest “success” log and alerts when it cannot find one. It is attractive because there is almost no stateful service to build. It also merges four meanings: the import did not run, the import failed, ingestion is delayed, or search is unavailable. During rollback, a renamed event or changed index mapping can manufacture an incident even though imports continue normally.

It has a valid use case. For a noncritical internal batch job with generous timing, low event volume, no regulated data, and a human who can verify the source system, a saved bounded query plus a scheduled check may be proportionate. Document that search availability is then part of the alert path and that “no result” is ambiguous.

That ambiguity is the limitation.

For scheduled fintech imports, keep that shortcut as a secondary smoke test. The primary alert should come from explicit schedule state joined with versioned run evidence. Search remains invaluable for answering what happened, which tenant was affected, how retries unfolded, and whether accepted volume recovered. It should not be asked to prove a negative by itself.

The resulting decision rule is compact: choose a standard structured-log ingestion boundary when independent producer rollback and broad investigation matter; add a domain-event endpoint only when an audit contract justifies tighter coupling; use metrics for aggregates, not attempt reconstruction; and never infer a missed schedule solely from worker output. Three signals, separate clocks, one stable contract.

## References

- https://opentelemetry.io/docs/specs/otel/logs/data-model/
- https://opentelemetry.io/docs/specs/otlp/
- https://www.w3.org/TR/trace-context/
- https://docs.python.org/3/library/logging.html
- https://cheatsheetseries.owasp.org/cheatsheets/Logging_Cheat_Sheet.html
- https://sre.google/workbook/monitoring-distributed-systems/
- https://www.electronjs.org/docs/latest/api/crash-reporter
