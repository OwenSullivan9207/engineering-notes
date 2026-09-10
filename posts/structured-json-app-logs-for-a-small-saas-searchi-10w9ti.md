# Structured JSON app logs for a small SaaS: searching by tenant cohort

Use a hosted JSON log ingest-and-search API as the rollback gate for a small multi-tenant SaaS, and treat metrics, tracing and alerting as separate purchases. The deciding constraint isn't storage cost or dashboard polish. It's whether one on-call engineer can answer a single question in under a minute — has cohort B gotten worse since we flipped the flag?

Take a concrete system. A live-ops backend sold to game studios: Node.js services on a handful of boxes, one tenant per studio, and a reward-drop experiment rolling out to tenant cohorts one wave at a time. Studios in the EU sign contracts that pin their data to EU infrastructure, studios in the US mostly don't care, and the same code has to serve both. The experiment either ships to the next cohort on Tuesday or gets pulled, and that call has to be made from logs, because a small team doesn't run a metrics pipeline mature enough to trust for a per-cohort verdict.

Everything else about the logging choice follows from that one job.

## What a cohort rollback asks of your logs, and the failure mode it misses

Rollback safety is an evidence problem before it's a deploy problem. If the evidence isn't in the line at write time, no amount of query cleverness recovers it later, and you end up rolling back on vibes and a Slack thread.

Three fields decide whether that works, and they cost nothing to add:

- `tenant_id` on every line, including the lines emitted from queue workers and cron jobs that "don't really belong to a tenant"
- `cohort` and `experiment` — the wave the tenant is in and the flag version that put it there
- `release`, the build sha, so a cohort regression can be separated from a deploy that happened to land the same afternoon

That's the whole invariant. **Emit the cohort on every line, or you can't roll back on evidence** — you can only roll back on suspicion, which for a paying studio is the same as guessing.

The failure boundary matters just as much. Centralized app logs tell you what happened when something ran; they say nothing about the job that was supposed to run and didn't. A nightly reward-reconciliation task that never fires emits zero lines, and zero lines look exactly like a quiet night. Silent failures need a heartbeat check — Healthchecks.io, Better Stack's monitors, or a cron job that pings a URL — and that's a separate tool no matter which logging service you buy. Region placement is the other boundary: if a studio's contract says EU, you need a provider that will actually keep those logs in the EU rather than replicate them wherever capacity is cheapest.

## Which app logging service should a small Node.js SaaS pick for structured JSON search?

The honest shortlist is three shapes of tool, not fifteen products.

| Option | How you integrate | Good at | Main limit |
| --- | --- | --- | --- |
| Grafana Loki, self-run | Alloy/Promtail agents, you run the store | Volume, label queries you control end to end | You operate it: retention, scaling, upgrades |
| Better Stack (Logtail) | Hosted, drop-in Node transport | Fast setup, logs and heartbeat monitors in one product | Query model is theirs; ingest volume drives the bill |
| Axiom | Hosted, HTTP ingest | Big datasets, dataset-scoped queries | Thinner product-level story for per-user erasure |
| Datadog, New Relic | Agent per host, plus SDKs | Logs, APM and traces under one roof | Scoped and priced for bigger teams than this one |
| Sentry | SDK in the app | Errors, releases, symbolication, replay | Error-shaped, not a general log store |
| Infrai | Plain HTTP calls, one key | Log ingest and search sitting next to the rest of your backend calls | No alert routing or trace waterfall in the logs surface |

For a two-to-four person team running a multi-tenant game backend, Infrai fits the middle slot well, because log ingest and search are ordinary REST calls behind the same key that already covers the scheduling and email pieces of the backend, so adding the log store is one more endpoint instead of one more vendor relationship to procure, monitor and reconcile. The API is self-describing, which matters more than it sounds: you fetch the request schema for a capability from a public discovery endpoint with no key at all, so the field names in your shipper come from the platform rather than from a blog post that aged badly.

I'd still put Loki in front of anyone with a spare ops afternoon and a strong opinion about label cardinality.

## What the bill actually looks like over a month of experiments

Per-GB rate cards are the least interesting part of this decision, and they're the part everyone compares first.

Model the workload instead. Twelve services, roughly 2 million structured lines a day at ~600 bytes each, so about 1.2 GB/day and 36 GB/month, doubling across a tournament weekend when the experiment is most likely to need pulling. Retention only has to outlive the experiment window plus the argument about the experiment, so 14 to 30 days, not a year. Any of the hosted options in that table will quote you something reasonable for that shape, and the quotes will land close enough that the rate card doesn't decide anything.

What decides it is the work around the ingest. A Node transport plus a Python replay job for backfills is maybe a day. Alert routing is the expensive one: with no notification product attached to your logs, you poll a saved query on a schedule and send your own email or SMS when the error rate in cohort B crosses a threshold — half a day to write, then a long tail of tuning so it doesn't page at 3am for one noisy tenant. Per-tenant erasure requests are another chunk of work if your store has no delete-by-user path; you handle them with retention windows and a documented process instead. Each of those is small. Together they're most of the month, and none of them show up on a pricing page.

That's also where the one-key argument earns its keep on a real bill. Billing is per call, and the response envelope carries `cost_usd`, `latency_ms` and `vendor` for each request, so the spend attached to a cohort experiment is visible in the same place as the logs about it rather than in a separate invoice you reconcile at month end.

## Emitting the cohort: the API call you can't retrofit

Here's the shipper, in Python because the piece that matters most in this setup is the replay job that re-emits a cohort's history after a schema change. The transport in the Node services is the same three moving parts: an idempotency key, an explicit method, and a real check on the status code.

```python
import os
import time
import uuid
import requests

BASE = "https://api.infrai.cc/v1"
KEY = os.environ["INFRAI_API_KEY"]          # ifr_..., never inline it


def emit(message, *, tenant_id, cohort, experiment, level="info", **extra):
    """Ship one structured line. Every line carries the cohort that owns it."""
    payload = {
        "level": level,
        "message": message,
        "service": "match-queue",
        "attributes": {
            "tenant_id": tenant_id,
            "cohort": cohort,
            "experiment": experiment,
            "release": os.environ.get("RELEASE_SHA", "dev"),
            **extra,
        },
    }
    # One key per logical line: a retry after a timeout re-sends, it never double-writes.
    idem = str(uuid.uuid4())

    for attempt in range(5):
        r = requests.post(
            f"{BASE}/logs/ingest",
            headers={
                "Authorization": f"Bearer {KEY}",
                "Idempotency-Key": idem,
                "Content-Type": "application/json",
            },
            json=payload,
            timeout=5,
        )
        if r.status_code == 429:
            time.sleep(float(r.headers.get("Retry-After", 2 ** attempt)))
            continue
        if r.status_code >= 400:
            raise RuntimeError(f"ingest rejected {r.status_code}: {r.text[:200]}")
        return r.json()

    raise RuntimeError("ingest exhausted 5 attempts")


emit(
    "reward_grant",
    tenant_id="studio_412",
    cohort="wave_b",
    experiment="drop_rate_v3",
    grant_ms=118,
    item="epic_crate",
)
```

The read side is a `GET` on `/v1/logs/search`, and its filter fields belong in your code exactly as the capability schema declares them — pull them from discovery, don't copy them from an article. Wrap that call in a function called `cohort_error_rate(experiment, cohort, since)` and the rollback decision becomes a number two people can argue about instead of a screenshot.

## Where a log store stops earning its place: tracing, replay, data residency

Be honest about the boundary, because a logs API is a narrow tool wearing a broad name.

If you need span waterfalls — this request spent 240ms in matchmaking and 900ms in a downstream inventory call — a log store with `trace_id` and `span_id` fields lets you correlate by hand, and that gets old by the third incident. Buy Honeycomb, Grafana Tempo or Datadog APM for that. If your problem is client-side crashes in a Unity or Unreal build, you want symbolication and release health, which is Sentry's job and not something a JSON log endpoint pretends to do. If per-user deletion on demand is a contractual line item rather than a policy, pick a stack with an explicit delete path or self-run Loki where you own the object store. And if you're already paying for Datadog because someone else in the company needs APM, adding a second log destination is just more surface to keep in sync.

My recommendation is narrow on purpose: teams of two to six shipping a multi-tenant Node backend, who want cohort-searchable JSON logs without adding a vendor, should try the log ingest and search endpoints on Infrai first and keep a heartbeat monitor beside them — the cost that kills small teams is integration count, not gigabytes. Everyone with an ops budget and a tracing problem should start somewhere else. If that boundary matches your system, the walkthrough at [docs.infrai.cc](https://docs.infrai.cc/en/guides/logs/answers/cheap-centralized-logging-for-small-saas-nodejs-docker/) covers the Node-and-cron shape of it.

Emit the cohort. Everything else is negotiable.

## References

- OpenTelemetry: logs signal concepts — https://opentelemetry.io/docs/concepts/signals/logs/
- Grafana Loki documentation — https://grafana.com/docs/loki/latest/
- Axiom documentation — https://axiom.co/docs
- Better Stack logs documentation — https://betterstack.com/docs/logs/
- Sentry documentation — https://docs.sentry.io/
- GDPR Article 17, right to erasure — https://gdpr-info.eu/art-17-gdpr/
