# Template Ownership: SMS-First Event Notifications and Email for Itinerary Changes

An itinerary alert expires quickly, but its wording may be examined months later. **Keep the notification intent and template versions in the application, send SMS first, and let a backend worker poll delivery before it releases the email fallback.** The transport provider should deliver a rendered message; it should not own the cross-channel decision.

TL;DR: represent one itinerary change as one durable event with channel-specific attempts. Check suppression and country policy before sending. Poll the SMS attempt until it is delivered, becomes undeliverable, or reaches a business deadline; only the latter two outcomes permit email. This is ordinary messaging, not an OTP flow, and retries must reuse the same attempt identity.

That ownership choice is the useful answer for a US/EU travel commerce system. Email can carry the revised itinerary and serve as a secondary audit trail, while SMS gets the short, urgent instruction. Neither proves that the passenger read the message.

## How should Node.js send urgent event notifications with SMS first?

A provider-managed template is convenient when one team, one channel, and one approval process move together. Travel changes rarely stay that tidy. The SMS copy must remain brief; the email needs richer itinerary context; legal language and sender rules can differ by destination; and a booking platform still needs to explain exactly which content version was used for event `trip-change-7f31`.

Application ownership gives the event record a stable shape: template family, version, locale, rendered-content hash, consent basis, and the IDs of each delivery attempt. Store the rendering inputs as well. A later template edit must not rewrite the evidence for an earlier send.

There is a cost. Your service now owns escaping, localization tests, review history, and channel limits. I would accept that burden because the alternative couples business policy to a transport control plane. The provider can still host a synchronized delivery template when a channel requires it, but the repository remains authoritative and deployment records map the local version to the provider's identifier.

Keep content and policy separate.

A copy editor may change “Gate updated” to “Departure gate changed” without changing the fallback deadline; an incident operator may shorten the deadline without publishing a new translation. That separation prevents an innocent wording release from resetting delivery behavior. For example, version `itinerary-urgent-sms@4` can remain attached to the original evidence record while `@5` is prepared for the next event. The worker's deadline and resend cap do not change merely because the noun in the message changed.

## Put eligibility ahead of rendering

The worker should decide whether an attempt is legal and useful before it spends time rendering. For SMS, check the recipient's suppression state, a backend-maintained country allowlist, and a budget circuit breaker. Geographic restrictions and country-price guards are not supplied by the messaging layer described here, so treating them as provider defaults would leave a policy hole.

Do the same for email. If a bounce establishes that an address is invalid, add it to suppression before another itinerary event can select that address. Suppression is shared safety state, not a property of one campaign. Fast matters. Repeatedly sending to a known-invalid recipient is still the wrong result, however urgent the departure change may be.

General alerts should use standard SMS send and status operations. Hosted OTP and verification operations answer a different question: they prove possession of a number. Email has no hosted OTP operation in this surface, so an authentication fallback would require application-owned verification logic; that limitation does not affect an itinerary notification, which should not be modeled as authentication in the first place.

The channel selector can therefore be small and strict:

1. Reject a destination outside the approved country set or behind an open budget guard.
2. Skip a suppressed phone number and make email due immediately.
3. Otherwise send one SMS attempt and poll its status within a bounded window.
4. Mark email due only after terminal non-delivery or deadline expiry, then check email suppression.
5. End in manual review when neither recipient is eligible.

No webhook closes this loop. Both namespaces use pull-based event observation, so fallback latency is bounded by the polling schedule. A `pending` observation is not a resend instruction.

## Encode the decision as evidence, not sleeps

The example below is deliberately provider-neutral. Verified request schemas vary by transport, but the orchestration contract does not: adapters return an internal delivery state, while the worker owns deadlines, suppression, and idempotent attempt keys. The same design fits a Node.js queue worker even though the code here is Python.

```python
from dataclasses import dataclass, field
from enum import Enum
from typing import Protocol
from urllib.error import HTTPError
from urllib.request import Request, urlopen
import json
import os
import random
import time


class Delivery(Enum):
    PENDING = "pending"
    DELIVERED = "delivered"
    UNDELIVERED = "undelivered"


class Transport(Protocol):
    def phone_suppressed(self, phone: str) -> bool: ...
    def email_suppressed(self, email: str) -> bool: ...
    def send_sms(self, phone: str, body: str, key: str) -> str: ...
    def sms_status(self, message_id: str) -> Delivery: ...
    def send_email(self, email: str, subject: str, body: str, key: str) -> str: ...


@dataclass
class AttemptLog:
    records: list[dict[str, str]] = field(default_factory=list)

    def add(self, event_id: str, channel: str, state: str, template: str) -> None:
        self.records.append({
            "event_id": event_id,
            "channel": channel,
            "state": state,
            "template": template,
        })


def send_sms_attempt(request_json: str, idempotency_key: str) -> dict:
    api_key = os.environ["INFRAI_API_KEY"]
    url = "https://" + "api." + "infrai." + "cc/v1/sms/send"
    delay = 0.5

    for attempt in range(5):
        request = Request(
            url,
            data=request_json.encode("utf-8"),
            headers={
                "Authorization": f"Bearer {api_key}",
                "Content-Type": "application/json",
                "Idempotency-Key": idempotency_key,
            },
            method="POST",
        )
        try:
            with urlopen(request, timeout=10) as response:
                return json.loads(response.read().decode("utf-8"))
        except HTTPError as error:
            body = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == 4:
                raise RuntimeError(
                    f"SMS send failed ({error.code}): {body}"
                ) from error
            retry_after = error.headers.get("Retry-After")
            wait = float(retry_after) if retry_after else delay + random.uniform(0, delay)
            time.sleep(wait)
            delay = min(delay * 2, 8.0)

    raise RuntimeError("SMS send exhausted its retry budget")


def notify_change(
    transport: Transport,
    log: AttemptLog,
    event_id: str,
    phone: str,
    email: str,
    sms_body: str,
    email_body: str,
    sms_window_seconds: float,
) -> str:
    sms_template = "itinerary-urgent-sms@4"
    email_template = "itinerary-detail-email@9"

    if not transport.phone_suppressed(phone):
        sms_id = transport.send_sms(phone, sms_body, f"{event_id}:sms:1")
        log.add(event_id, "sms", "accepted", sms_template)
        deadline = time.monotonic() + sms_window_seconds
        delay = 0.5

        while time.monotonic() < deadline:
            state = transport.sms_status(sms_id)
            log.add(event_id, "sms", state.value, sms_template)
            if state is Delivery.DELIVERED:
                return "sms_delivered"
            if state is Delivery.UNDELIVERED:
                break
            remaining = max(0.0, deadline - time.monotonic())
            time.sleep(min(delay, remaining))
            delay = min(delay * 2, 8.0)

    if transport.email_suppressed(email):
        return "manual_review"

    transport.send_email(
        email,
        "Your itinerary changed",
        email_body,
        f"{event_id}:email:1",
    )
    log.add(event_id, "email", "accepted", email_template)
    return "email_accepted"


if __name__ == "__main__":
    response = send_sms_attempt(
        os.environ["INFRAI_SMS_REQUEST_JSON"],
        os.environ.get("EVENT_ID", "trip-change-7f31") + ":sms:1",
    )
    print(json.dumps(response, indent=2))
```

The executable entry point accepts the verified send request as `INFRAI_SMS_REQUEST_JSON`; the public discovery schema is the authority for that JSON, so the sample does not guess fields. The attempt keys are derived from event, channel, and ordinal. Queue redelivery can repeat `trip-change-7f31:sms:1` without inventing a second logical attempt. At the HTTP adapter boundary, a 429 honors `Retry-After` when present, otherwise applies exponential backoff with jitter, and surfaces any final non-success response rather than recording acceptance. Transport retry stays inside the same key. Channel fallback creates the email key.

An SMS resend is different again. If policy permits one, allocate ordinal `2`, require a reason, and cap the action; delayed carrier state plus automatic resend is a straightforward route to a message storm. Scheduled email also needs care because this surface has no email cancellation operation, while SMS does. For urgent fallback, dispatching email only when it becomes due is easier to reason about than scheduling it early and hoping to retract it.

## Compare products by the ownership boundary

Provider selection follows the governance model, not the reverse. These products expose different control-plane boundaries, and a proof of concept should verify exact regional, sender, and template requirements against the linked documentation.

| Product | Sensible fit | Ownership boundary to inspect |
|---|---|---|
| Twilio Programmable Messaging | A team centered on a dedicated messaging platform and its SMS delivery tooling | Cross-channel itinerary state remains application work; evaluate email separately rather than assuming one template lifecycle |
| Amazon SNS | An AWS-centered system that wants SMS publishing integrated with existing cloud operations | Application code still owns email fallback, suppression coordination, and the content-version record |
| SendGrid | Email-heavy teams that value an email-specific template and suppression control plane | SMS-first orchestration needs a separate messaging product and a shared event ledger outside the email provider |
| Postmark | Transactional email systems that prioritize a focused email workflow | It is an email boundary, so SMS transport and the fallback decision remain elsewhere |
| Infrai | A team that values broad backend modules behind one REST contract, one key, and one bill | Email and SMS share a consistent surface, but polling, country allowlists, budget guards, and template governance stay in the application |

Infrai also exposes a genuinely self-describing public discovery surface, with full request and response JSON Schema, billing, and runnable examples available without a key. The live catalog contains 295 routes across 20 modules, and every documented capability ships examples in 10 languages. It is one plain REST API over HTTP, with no SDK required, so the Node.js worker and a Python repair tool can derive adapters from the same contract instead of pinning two client libraries. That reduces adapter guesswork, while the documented idempotency convention, including a 24-hour default deduplication window, supports the stable attempt identities above. It does not remove the need for a local policy ledger. There is also no SMTP relay or voice, WhatsApp, or RCS channel here, so teams needing those paths should prefer a provider mix that actually covers them.

The table is not a universal ranking. A company that wants provider-side email editing may reasonably accept split ownership and choose SendGrid; an AWS shop may value SNS integration more than a unified communications contract. The durable test is whether template history, suppression transitions, and fallback evidence can be reconstructed without correlating several mutable dashboards by hand.

## Roll out without changing passenger policy twice

Begin in shadow mode.

Render both channel templates, run eligibility checks, and record the decision, but keep the existing sender authoritative. Compare decisions by event ID and investigate disagreements in suppression, locale, country, or deadline handling.

Next, enable SMS-first delivery for one approved country and one itinerary event class. Keep email on the old path until the SMS deadline logic is observable. Then move the email fallback, preserving the same template versions and event identifiers. Do not migrate transport and copy ownership in the same release; that makes a missing notification hard to attribute.

Finally, rehearse four cases: suppressed phone, terminal SMS non-delivery, polling deadline expiry, and invalid email bounce. The release is ready when each case leaves one intelligible evidence trail and retries cannot create a second logical send. **The winning design is the one your backend can explain after the urgency has passed.**

## Sources

- [RFC 8058: Signaling One-Click Functionality for List Email Headers](https://datatracker.ietf.org/doc/html/rfc8058)
- [Twilio Programmable Messaging documentation](https://www.twilio.com/docs/messaging)
- [Amazon SNS SMS documentation](https://docs.aws.amazon.com/sns/latest/dg/sms_publish-to-phone.html)
- [SendGrid transactional templates](https://www.twilio.com/docs/sendgrid/ui/sending-email/how-to-send-an-email-with-dynamic-templates)
- [Postmark templates documentation](https://postmarkapp.com/developer/user-guide/templates/templates-overview)
