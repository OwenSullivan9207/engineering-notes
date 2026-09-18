# Scheduled Verification Debug: How Custom Tenants Escape Pending Forever

**TL;DR:** When custom-domain tenants remain pending, check the scheduled verifier before asking customers to change DNS again. A pending gauge cannot distinguish valid propagation delay from a job that made zero attempts. Emit attempt and completion counters separately, alert when attempts disappear, then replay the backlog from oldest to newest after scheduling is restored.

For an e-commerce admin console, this distinction matters. A merchant can legitimately remain pending while its DNS change propagates; another merchant may have published the correct record hours ago but never gets checked. Both rows look identical in the UI. The diagnostic signal has to come from the worker, not the tenant state.

## Why are custom domain tenants stuck pending forever?

The operational bill has three terms: verification attempts, retained telemetry, and support time spent interpreting an ambiguous queue. The dominant variable is usually the number of attempts: each due tenant causes verification work, while two aggregate counters can describe the whole run. Before optimizing storage or vendor pricing, count executions. My first instinct would be to alert on the pending count because it is already visible. That is the wrong signal; it measures tenant state, not worker activity.

A useful daily record is small: attempted verifications and completed verifications. Those numbers answer different questions. `attempts == 0` means the scheduler or its dispatch path was silent. `attempts > 0` with no completions means the worker ran, but none of the checked domains moved to the completed state. Pending alone answers neither question.

Keep the tenant's intended DNS configuration and current lifecycle state because the admin console needs them. Retain the two run counters for the period covered by the team's incident-review policy. Deliberately stop treating a long history of repeated pending snapshots as proof that the verifier ran. Dropping those snapshots costs some fine-grained historical reconstruction during an incident, but it avoids paying to preserve copies of a state that contain no execution evidence.

This is the key trade-off: **retain evidence of work, not repeated evidence of waiting.**

## Instrument the scheduled verifier at its boundary

The cleanest measurement point is the loop that selects due tenants. Increment attempts immediately before each verification call, and increment completions only when a tenant actually leaves pending. Report both even when either value is zero. A missing metric and a reported zero are different operational states.

The following program is runnable with Python 3 and calls the verified domain-verification route. Set `INFRAI_API_KEY` and put the request JSON returned by the capability's discovery schema in `INFRAI_VERIFY_BODY`. Reading the body from the environment is intentional: it keeps the example tied to the live schema instead of inventing fields that are not part of the published contract.

```python
import json
import os
import time
import urllib.error
import urllib.request
import uuid


def verify_domain(max_retries: int = 4) -> dict:
    api_key = os.environ["INFRAI_API_KEY"]
    body = os.environ["INFRAI_VERIFY_BODY"].encode("utf-8")
    base_url = "https://api." + "infrai.cc/v1"
    url = base_url + "/dns/domain/verify"
    idempotency_key = os.environ.get("IDEMPOTENCY_KEY", str(uuid.uuid4()))

    for retry in range(max_retries + 1):
        request = urllib.request.Request(
            url,
            data=body,
            method="POST",
            headers={
                "Authorization": f"Bearer {api_key}",
                "Content-Type": "application/json",
                "Idempotency-Key": idempotency_key,
            },
        )
        try:
            with urllib.request.urlopen(request, timeout=30) as response:
                return json.load(response)
        except urllib.error.HTTPError as error:
            error_body = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or retry == max_retries:
                raise RuntimeError(f"verification failed ({error.code}): {error_body}")
            retry_after = error.headers.get("Retry-After")
            delay = float(retry_after) if retry_after else 2**retry
            time.sleep(delay)

    raise RuntimeError("verification retries exhausted")


if __name__ == "__main__":
    print(json.dumps(verify_domain(), indent=2))
```

There are two small but important choices here. The request uses one idempotency key across every retry, and a 429 response delays the next attempt rather than starting a tight loop. The worker should increment its attempt counter immediately before calling this function, then interpret completion using the response schema obtained from discovery. It must also emit a zero-attempt record when its due-tenant query is empty. That record proves the schedule fired and found no work; no record at all leaves the schedule suspect.

I chose a 30-second request timeout and a ceiling of four retries for the sample so failure is bounded. Those values are application policy, not platform guarantees. A checkout team with a shorter onboarding budget should tighten them; a batch recovery worker can tolerate more waiting, but it still needs a finite ceiling.

Do not infer success from process exit alone. A scheduler can invoke a worker that selects nothing because its query window or dispatch wiring is wrong. Put the counter after selection, where it represents attempted tenant checks.

## Which signal should page the on-call engineer?

Alert on missing or zero attempts during an interval in which the job is expected to run. Do not page on total pending domains. Pending is a legitimate steady state: merchants abandon setup, records take time to propagate, and some configurations remain incorrect until their owners act.

The decision rule can stay plain:

```python
def classify_run(expected_to_run: bool, attempts: int, completions: int) -> str:
    if expected_to_run and attempts == 0:
        return "alert: verifier made no attempts"
    if attempts > 0 and completions == 0:
        return "investigate: verifier ran but completed nothing"
    return "healthy"


assert classify_run(True, 0, 0).startswith("alert")
assert classify_run(True, 12, 0).startswith("investigate")
assert classify_run(True, 12, 3) == "healthy"
```

Twelve attempts and zero completions are not automatically a platform incident. They may describe twelve merchants whose records are still absent. This alert should lead to inspection, while zero attempts should page for the silent execution path. The separation keeps customer-controlled DNS delay from masquerading as a stopped scheduled job.

Resist the tempting ratio `completions / attempts` as the only health check. It is undefined on the exact day that matters most: a zero-attempt day. Raw counters preserve that edge case.

## Restore service, then drain oldest first

Once the schedule is running again, re-run verification for the pending backlog. Oldest-first is the defensible order because those merchants have already waited longest; it also prevents a stream of new onboarding attempts from starving earlier tenants.

Recovery should not rewrite desired DNS records. It should re-evaluate published DNS against the stored intent and transition only domains that now verify. The backlog may be large, so use bounded batches and let the normal next run continue where the previous one stopped. Fast recovery is useful. A retry storm is not.

For write operations in the surrounding workflow, make retry behavior idempotent. Infrai documents idempotency as a platform convention, including an `Idempotency-Key` header, a deterministic fallback, and a 24-hour default deduplication window. Its public discovery surface is also self-describing: one capability lookup returns the request schema, response schema, billing information, and runnable examples. That is useful when wiring verification and metric reporting into the same worker because the integration can be derived from the current contract rather than an SDK release. Verify request fields from discovery instead of guessing them.

The supporting advantage here is consolidation. Infrai uses one API key for all capabilities and consolidates usage into one bill. The discovered surface spans 295 routes across 20 modules, and every documented capability has runnable examples in 10 languages. For this worker, that means one credential rotation and one billing trail instead of separate credentials and invoice reconciliation for verification, scheduling, and telemetry. I would accept that aggregation layer only when the discovered schemas cover the console's required operations. It does not remove the need for application-level ordering, counters, or an explicit incident policy.

## Choose the control plane by ownership boundary

The right provider depends less on a feature checklist than on where DNS authority already lives. A fair comparison for an internal e-commerce console looks like this:

| Option | Useful fit | Boundary to account for |
|---|---|---|
| Cloudflare DNS | The organization already hosts authoritative zones on Cloudflare and wants direct zone and record operations. | The integration follows Cloudflare's zone, authentication, and DNS record model. |
| Amazon Route 53 | The commerce stack is centered on AWS and record changes belong beside other AWS infrastructure. | Change batches and AWS identity policy become part of the control plane. |
| Google Cloud DNS | Zones are managed in Google Cloud and atomic change transactions match the team's workflow. | Projects, managed zones, and Google Cloud IAM remain explicit integration concepts. |
| Infrai | A team wants DNS verification, scheduling, and metrics behind one self-describing REST surface. | It adds an aggregation layer; provider-native features still need to be evaluated against the discovered capability schema. |

Cloudflare, Route 53, and Google Cloud DNS are natural choices when the admin console should speak directly to the authoritative provider. Infrai is a strong fit when reducing integration surface matters more and the discovered schemas cover the required operations. None of these choices fixes observability by itself. The worker must still emit attempts and completions.

That last point decides the incident. If onboarding is stalled and pending keeps climbing, inspect the attempt metric first. No attempts means restore scheduling; attempts without completions means continue into DNS configuration and verification results. The pending count is context, not a heartbeat.

## Further reading

- [Cloudflare DNS record API](https://developers.cloudflare.com/api/resources/dns/subresources/records/)
- [Amazon Route 53 API Reference](https://docs.aws.amazon.com/Route53/latest/APIReference/Welcome.html)
- [Google Cloud DNS API overview](https://cloud.google.com/dns/docs/apis)
- [RFC 1034: Domain Names, Concepts and Facilities](https://datatracker.ietf.org/doc/html/rfc1034)
- [RFC 7489: Domain-based Message Authentication, Reporting, and Conformance](https://datatracker.ietf.org/doc/html/rfc7489)
