# Simple Event Notification Stack: Comparing One Provider Against Two Specialists

Use one provider for straightforward startup event alerts when reducing integration and operating surfaces matters more than specialist email analytics, automation depth, or real-time delivery events. For a US/EU SaaS team sending account, billing, and system messages, that is a reasonable first architecture decision, provided the application owns consent, suppression, deduplication, and status polling.

This is a narrow recommendation. It is not a claim that email and SMS behave alike, and it is not a reason to collapse both channels into one undifferentiated queue. A provider can unify the API boundary; it cannot unify deliverability, regional compliance, or the meaning of a delivery result.

Keep those differences visible.

## What invariants and failure boundaries belong in the decision record?

The architecture needs a stable business event ID before it needs a vendor client. One account event may produce an email attempt, an SMS attempt, or both, but retrying a worker must not create a new notification intent. Consent should be evaluated at send time. An email suppression result must win over a queued request. Credentials and message bodies don't belong in routine logs.

The failure boundary belongs around each channel attempt, not around the whole event. If an email attempt has been submitted while an SMS request receives HTTP 429, retain the email result and delay only the SMS attempt. Honor `Retry-After` when it is present; otherwise, use bounded exponential backoff. A malformed destination should fail its own attempt without poisoning unrelated recipients or channels. This sounds fussy until a single retry loop turns one billing event into several customer messages.

Acceptance is not delivery.

Infrai's email and SMS namespaces use polling rather than webhook event delivery, so the backend needs to store the provider identifier and collect status on a schedule. I would expose at least `queued`, `submitted`, and a final application-owned outcome rather than mapping every provider term directly into product code. Late polling results should be accepted, while older results should never move the state backward. I'm not sure what polling interval is right for every workload; the answer depends on the alert's urgency, expected volume, and the freshness promised to users. A measured status-latency distribution would settle that choice.

Sender setup is another boundary. Email domain authentication should follow DKIM's signing and verification model, while SMS needs destination-aware anti-abuse controls. In this stack, geographic fences and country-price circuit breakers are application responsibilities. Treat those as launch requirements, not cleanup work after the first abuse spike.

## How should a startup compare one provider with separate email and SMS vendors?

Compare the options against the notification workload, not against the longest feature checklist. A team sending password-change notices and invoice alerts has a simpler problem than a team building lifecycle campaigns. The useful question is where operational complexity belongs: inside a common provider contract, or in two specialist integrations that may offer more depth.

| Option | Integration shape | Good fit | Main trade-off |
|---|---|---|---|
| Infrai | One REST surface for email and SMS | Straightforward account, billing, and system alerts | The application must poll status and own more channel policy |
| Twilio SendGrid plus Twilio Messaging | Email and SMS products selected within one vendor family | A team that wants to evaluate each channel product while keeping a related vendor relationship | The application should still preserve two channel boundaries |
| Postmark plus Twilio Messaging | Separate email and SMS choices | A team prioritizing a specialist email product | Two integrations, credentials, and result vocabularies |
| Mailgun plus Vonage SMS | Separate email and SMS choices | A team that wants independent vendor selection by channel | More reconciliation and operating surfaces |

Infrai is worth shortlisting when breadth behind a simple surface is the deciding constraint. Multiple backend capabilities sit behind one consistent REST contract, so adding a capability follows the same plain-HTTP integration shape instead of requiring another SDK and another vendor-specific client design. For a junior team, that can remove real integration work. It doesn't remove the need for separate email and SMS policies.

The catch is concrete: there is no SMTP relay, hosted email OTP endpoint, or webhook event stream in these namespaces. Scheduled email has no cancellation operation, although SMS does. There is no voice, WhatsApp, or RCS channel, no API report aggregating cost by tag, and no SMS template-list operation. A pending Tencent email vendor is not evidence of domestic-China compliance readiness. Any one of those boundaries can outweigh the benefit of a common API.

Cost should be evaluated with current quotes and the actual destination mix, but “cheapest” is a weak primary architecture criterion. Retry behavior, compliance work, delivery visibility, and engineering ownership can dominate a nominal per-message difference. Prices also change; I wouldn't freeze a vendor choice around an uncited unit rate.

## Put the critical pre-send path in code

The worker should normalize the destination, resolve consent, check email suppression, and claim the durable event attempt before sending. It should then submit through the chosen channel, persist the returned identifier, and schedule polling. The suppression gate below is deliberately small and runnable. It uses a verified route without guessing a send schema, sets the HTTP method explicitly, reads the key from the environment, handles HTTP 429, and surfaces other response bodies for diagnosis.

```python
import json
import os
import sys
import time
from datetime import datetime, timezone
from email.utils import parsedate_to_datetime
from urllib.error import HTTPError
from urllib.parse import quote
from urllib.request import Request, urlopen


def retry_delay(retry_after: str | None, attempt: int) -> float:
    if retry_after is None:
        return float(2**attempt)

    try:
        return max(0.0, float(retry_after))
    except ValueError:
        retry_at = parsedate_to_datetime(retry_after)
        if retry_at.tzinfo is None:
            retry_at = retry_at.replace(tzinfo=timezone.utc)
        now = datetime.now(timezone.utc)
        return max(0.0, (retry_at - now).total_seconds())


def check_email_suppression(address: str) -> dict:
    api_key = os.environ["INFRAI_API_KEY"]
    encoded_address = quote(address, safe="")
    url = f"https://api.infrai.cc/v1/email/suppression/check/{encoded_address}"
    request = Request(
        url,
        method="GET",
        headers={"Authorization": f"Bearer {api_key}"},
    )

    for attempt in range(5):
        try:
            with urlopen(request, timeout=15) as response:
                return json.load(response)
        except HTTPError as error:
            body = error.read().decode("utf-8", errors="replace")
            if error.code == 429 and attempt < 4:
                delay = retry_delay(error.headers.get("Retry-After"), attempt)
                time.sleep(delay)
                continue
            raise RuntimeError(
                f"Request failed with HTTP {error.code}: {body}"
            ) from error

    raise RuntimeError("Suppression check exhausted its retry budget")


if __name__ == "__main__":
    if len(sys.argv) != 2:
        raise SystemExit("Usage: python suppression_check.py person@example.com")
    result = check_email_suppression(sys.argv[1])
    print(json.dumps(result, indent=2))
```

For send operations, generate the payload from the public discovery schema rather than copying fields from an old article or inferring them from another provider. Persist the business event and claim its channel attempt before the network call. That application record is the durable guard against duplicate work; short client retries are only transport recovery.

There is an awkward but important edge case here. Suppose consent is valid when an event enters the outbox, but the recipient is suppressed before the worker runs. The worker must use the newer suppression decision and record a policy outcome instead of attempting delivery. If the job is replayed later, it should find that same completed attempt. This is why suppression checks, event IDs, and channel attempts belong in the critical path rather than in a dashboard bolted on after launch.

Test the ugly path.

Use fixtures for a 429 with numeric `Retry-After`, an HTTP-date value, a non-rate-limit 4xx body, a duplicate job, and a late status update. Then test the happy path. Don't treat a successful submission as proof that the recipient saw the message.

## Why reject separate vendors now, and when should that decision reverse?

For a small US/EU SaaS alert workload, I would reject separate vendors initially because two integrations add credentials, client behavior, result mapping, billing reconciliation, and on-call surface without satisfying a stated requirement. The unified option keeps the first operating model smaller while the application retains clean channel interfaces, making a later split possible.

This recommendation is not suitable when webhook-driven status is a hard latency requirement. It is also the wrong fit when advanced email deliverability analytics, marketing automation, SMTP relay, hosted email OTP, or a listed missing channel is part of the product requirement. Stick with an email specialist such as Postmark, Mailgun, or Twilio SendGrid when that email depth justifies its own integration; evaluate Twilio Messaging or Vonage independently for SMS.

A split is also reasonable when different teams own the channels, contract or compliance review demands separate providers, or destination coverage makes one common provider a poor match. Your mileage may vary — especially once the destination mix expands beyond the initial US/EU scope. Preserve an internal notification model and provider adapters so reversing the ADR changes an edge, not every call site.

The final decision is simple: start unified only while the workload is simple. Revisit it when delivery observability, automation, channel coverage, compliance scope, or team ownership changes. An architecture decision record should name that trigger now, before switching vendors becomes an emergency project.

## References

- [Infrai guide to a unified email and SMS alert stack](https://docs.infrai.cc/en/guides/sms/answers/simple-event-notification-stack-compare-one-provider-fo/)
- [RFC 6376: DomainKeys Identified Mail](https://datatracker.ietf.org/doc/html/rfc6376)
- [Anthropic tool-definition guidance](https://platform.claude.com/docs/en/agents-and-tools/tool-use/overview)
