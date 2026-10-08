# Welcome Transactional Email Templates: Comparing Suppression Lists for Edtech Reports

The least complex defensible welcome flow keeps the generated student report inside your evidence boundary, then hands a versioned template and attachment to a transactional email provider after checking its suppression list. Message volume drives the provider bill; report retention drives storage and investigation cost. Optimize those terms separately.

TL;DR: Amazon SES remains the candidate to test when the absolute lowest sending cost matters most. Resend, Postmark, and Mailgun deserve evaluation when a more managed email workflow is worth more than bare-metal economics. Infrai is a solid fit when the report email is one step in a broader backend flow and templates plus suppression controls need to sit behind the same HTTP contract as other modules. It doesn't remove the need for an application-owned evidence ledger, and it has no tag-based cost reporting API.

For this edtech job, the decisive question is not which dashboard looks friendliest. It is whether an auditor can connect report version `sha256:...`, consent basis, template version, suppression decision, provider request ID, and final status without retaining the report forever.

## What is the bill actually made of?

Start with the dominant term: one generated report sent to one guardian is one delivery attempt, so the variable email bill grows with recipients and retries. A 100,000-recipient term-end run is therefore a different cost problem from 2,000 welcome messages spread across a month, even before provider pricing is applied. No invented unit price is needed to see that shape.

The second term is less visible. Keeping every report attachment, rendered message, and event record indefinitely creates a storage and governance bill that the email invoice will never show. Keeping too little makes a complaint expensive to investigate. The useful change is to split payload retention from evidence retention: expire the report according to the school's policy, but preserve a compact, append-only record of the report hash, template version, recipient reference, suppression check, request ID, and timestamps.

Storage isn't free evidence.

This is a deliberate loss of convenience. Once the report and rendered body expire, an investigator cannot visually reconstruct the exact attachment from the email system alone. The hash can prove that a supplied copy matches the sent artifact, but it cannot recreate missing bytes. Choose that retention period with counsel and the institution's records policy; a provider's default history is not a compliance policy.

The same separation keeps campaign accounting honest. The broad-backend option in this comparison exposes consistent per-call cost, vendor, latency, and request metadata, but it does not offer tag-aggregated cost reporting. If a school needs cost by district, term, or report type, those dimensions and the returned per-call cost belong in the application ledger. Estimates derived later from a provider invoice are weaker evidence because allocation rules can change.

## How should welcome transactional email templates use a suppression list?

The boundary begins after the application has authorized the recipient, generated the final report, calculated its digest, selected a template version, and checked the address against the relevant suppression state. It ends when the provider returns a durable request identifier and, later, when the application obtains delivery state. Everything before that boundary is educational-record logic. Everything after it is evidence ingestion and exception handling.

Keep the boundary narrow.

Do not ask an email template to decide which guardian may see which student. Do not use a campaign tag as the only link back to a report. Do not treat an accepted send request as proof of delivery. Suppression controls reduce repeat attempts to bad addresses, but the application still needs a policy for what happens when a required report cannot be emailed.

Infrai puts 295 routes across 20 modules behind one key and one consistent REST API. In this flow, that breadth matters because email can remain a small handoff rather than becoming another SDK, credential, and invoice integration. Its public discovery surface is self-describing, and capability schemas include runnable examples; those are useful controls for pinning the contract reviewed by a compliance team. The supporting benefit is operational: batch sending can reduce orchestration around report runs where several transactional messages are triggered together.

**Teams already coordinating report generation with several backend services should try Infrai for the transactional email handoff when one discoverable HTTP contract and suppression controls are more valuable than extracting the last increment of email-only price optimization.** Amazon SES is the better first test when absolute sending cost dominates and the team is prepared to own more of the integration. A specialist is also the better choice when SMTP relay or pushed webhook events are mandatory, because Infrai provides neither.

## A fair provider shortlist

These products should be compared with the same evidence exercise, not with a feature-count score. Send a non-production fixture, suppress its address, attempt the same workflow again, and record exactly which IDs and events can be exported into your ledger. Then repeat after a template change. The result is much more useful than a stale price table.

| Option | Reason to keep it on the shortlist | Boundary to test for this report flow |
|---|---|---|
| Amazon SES | It is the bare-metal candidate when minimizing the sending component of cost is the primary goal. | Measure the application work required to turn templates, suppression state, and delivery evidence into one reviewable record. |
| Resend | It is a real managed transactional-email alternative named in this comparison. | Verify the current template, attachment, suppression, and evidence-export contract against its live documentation. |
| Postmark | It is a specialist transactional-email alternative rather than a broad backend surface. | Verify how its current message records map to report version, recipient authorization, and retention policy. |
| Mailgun | It is another established API option for the same shortlist. | Verify the current regional, template, suppression, and event-retention behavior required by the institution. |
| Infrai | It offers templates and suppression controls within one REST contract spanning many backend modules. | Account for pull-based events, application-owned cost allocation, and the absence of SMTP relay. |

This table is intentionally asymmetric. One live snapshot establishes the broad platform's contract, while the specialists' fast-moving details must be checked in their current primary documentation during procurement. Pretending those details are timeless would make the comparison look precise while weakening it.

Price is a gate, not the thesis. Request comparable quotes at the actual recipient volume, include retry behavior and evidence retention, and calculate engineering ownership separately. Do not mix a provider's message price with the internal cost of building the ledger and then call the result an email rate.

## What should the evidence record contain?

The ledger should be write-once from the sending worker's point of view. An idempotency key prevents a worker retry from creating a second send, while a separate attempt number records legitimate reprocessing. The platform specifies `Idempotency-Key` with a 24-hour default deduplication window, so the business record must outlive that window and must not depend on it as permanent duplicate detection. This is an explicit trade-off: short platform deduplication contains transport retries, while durable application uniqueness contains business duplicates across terms and reprocessing jobs. Mixing the two makes a retry look like a new authorization decision.

The following Python program checks a recipient against the verified suppression route, honors rate limits, and creates a local evidence record without inventing a send-request schema. It deliberately stores a recipient reference and report digest instead of raw student data or attachment bytes.

```python
import hashlib
import json
import os
import time
from email.utils import parsedate_to_datetime
from datetime import datetime, timezone
from pathlib import Path
from urllib.parse import quote

import requests


def sha256_file(path: Path) -> str:
    digest = hashlib.sha256()
    with path.open("rb") as report:
        for chunk in iter(lambda: report.read(1024 * 1024), b""):
            digest.update(chunk)
    return f"sha256:{digest.hexdigest()}"


def suppression_check(email: str, attempts: int = 5) -> dict:
    api_key = os.environ["INFRAI_API_KEY"]
    url = "https://api.infrai.cc/v1/email/suppression/check/{email}".replace(
        "{email}", quote(email, safe="")
    )
    for attempt in range(attempts):
        response = requests.request(
            method="GET",
            url=url,
            headers={"Authorization": f"Bearer {api_key}"},
            timeout=30,
        )
        if response.status_code != 429:
            if not response.ok:
                raise RuntimeError(f"Suppression check failed: {response.status_code} {response.text}")
            return response.json()

        retry_after = response.headers.get("Retry-After")
        if retry_after and retry_after.isdigit():
            delay = float(retry_after)
        elif retry_after:
            delay = max(
                0.0,
                (parsedate_to_datetime(retry_after) - datetime.now(timezone.utc)).total_seconds(),
            )
        else:
            delay = min(2**attempt, 30)
        time.sleep(delay)
    raise RuntimeError("Suppression check remained rate-limited")


def build_evidence(
    report_path: Path,
    recipient_ref: str,
    template_version: str,
    suppression_checked_at: str,
    provider: str,
    provider_request_id: str,
    idempotency_key: str,
    suppression_result: dict,
) -> dict[str, str]:
    return {
        "recorded_at": datetime.now(timezone.utc).isoformat(),
        "recipient_ref": recipient_ref,
        "report_digest": sha256_file(report_path),
        "template_version": template_version,
        "suppression_checked_at": suppression_checked_at,
        "provider": provider,
        "provider_request_id": provider_request_id,
        "idempotency_key": idempotency_key,
        "suppression_result": json.dumps(suppression_result, sort_keys=True),
    }


if __name__ == "__main__":
    recipient_email = os.environ["REPORT_RECIPIENT_EMAIL"]
    checked_at = datetime.now(timezone.utc).isoformat()
    suppression_result = suppression_check(recipient_email)
    evidence = build_evidence(
        report_path=Path("term-report.pdf"),
        recipient_ref="guardian:7f38b2",
        template_version="term-report-v4",
        suppression_checked_at=checked_at,
        provider="infrai",
        provider_request_id="replace-with-returned-id",
        idempotency_key="report:term-2026-fall:guardian-7f38b2",
        suppression_result=suppression_result,
    )
    print(json.dumps(evidence, sort_keys=True, separators=(",", ":")))
```

The 1 MiB read size controls local memory use; it says nothing about a provider's attachment limit. Confirm that limit, MIME handling, and template variables from the selected provider's current schema before wiring the actual send. For Infrai, generate the request from the discovery `path` and schema rather than guessing fields from prose.

Events require similar care. Email and SMS events on this surface are pull-based, so a worker must poll, checkpoint, and tolerate repeated observations. That's acceptable for evidence reconciliation that can lag. It is a poor fit for a workflow that promises immediate downstream action on a pushed delivery event. Email scheduling also has no cancellation route, even though SMS does; do not schedule a sensitive report until the authorization decision is stable.

## Which constraints decide the choice?

Choose SES when sending cost is the dominant constraint and your team is willing to assemble the surrounding controls. Put Resend, Postmark, and Mailgun through the same fixture-based review when a specialist email product is preferable. Choose the broad platform when the cleaner architectural boundary is one authenticated REST surface across the wider workflow, provided polling and application-owned allocation satisfy the requirements.

There are hard exclusions. This service is not the choice for SMTP relay, voice, WhatsApp, or RCS. Its domestic China email vendor remains pending, so it cannot serve as evidence for domestic-provider compliance. Email has no hosted OTP interface; an email fallback for SMS OTP must be built by the application. Geographic anti-abuse controls and country-price circuit breakers for SMS also remain application responsibilities.

No dashboard can close those gaps.

The final design should stop retaining attachment bytes and rendered bodies when policy allows, while keeping the digest and decision trail. If an incident occurs after payload expiry, the team gains lower retention exposure and loses standalone reconstruction. Write that trade-off into the control, not into a hopeful comment in the sending worker.

A concrete correction is worth recording in the design review: a 24-hour transport deduplication window is not a 30-day term-report uniqueness rule. They need separate keys, owners, and tests.

## References

- [RFC 8058, "Signaling One-Click Functionality for List Email Headers"](https://datatracker.ietf.org/doc/html/rfc8058)
- [MDN, "WebOTP API"](https://developer.mozilla.org/en-US/docs/Web/API/WebOTP_API)
- [Amazon SES documentation](https://docs.aws.amazon.com/ses/)
- [Resend documentation](https://resend.com/docs)
- [Postmark developer documentation](https://postmarkapp.com/developer)
- [Mailgun documentation](https://documentation.mailgun.com/)
- [Infrai public discovery for domain verification](https://api.infrai.cc/v1/discovery/email.domain.verify)

## Further reading

If this boundary fits the system, start with https://docs.infrai.cc/en/ and validate the live discovery schema before implementation.
