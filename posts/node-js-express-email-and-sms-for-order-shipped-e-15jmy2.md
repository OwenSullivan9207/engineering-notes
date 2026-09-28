# Node.js Express Email and SMS for Order Shipped Event Notifications

A password-reset message has a short useful life. If an Express request waits for both email and SMS providers, a slow provider becomes user-facing latency; if a worker retries blindly, one click can become three codes in the inbox. **Short answer:** publish one job per recipient and channel, render a versioned template, claim an idempotency key in the database, and let a worker retry transient failures before moving exhausted jobs to a dead-letter queue. Use polling when delivery state matters, because this capability does not provide webhook event subscriptions.

The same shape works for an `order_shipped` event. The expiry policy changes, but the boundary does not: the request path records intent, while workers own external delivery. That boundary is the practical choice for a Node.js marketplace where integration effort matters more than accumulating provider-specific features.

## How should Express send an order shipped event notification by email?

The domain transaction should decide that a notification is needed. It should not promise that a carrier accepted it. Inside the transaction that updates a reset token or shipment, write an outbox record containing the event ID, recipient ID, channel, template version, and expiry time. A dispatcher then enqueues separate email and SMS jobs.

Keep those jobs separate. Email and SMS have different suppression rules, payload limits, and failure modes, and tying them into one retry unit causes a successful email to be sent again because SMS timed out. A per-channel key such as `password_reset:evt_7f31:user_42:email:v3` gives the database a stable uniqueness boundary. The provider's idempotency convention is useful too, but the database key remains the durable record when a worker restarts or the queue redelivers.

Expiry is part of correctness, not decoration. A worker that receives a password-reset job after its `expires_at` should mark it expired without sending. For `order_shipped`, a late message may still have value, so that event can use a longer policy. Do not bury both decisions in a generic retry count.

Fail fast.

## The worker contract and retry boundary

The worker needs a small state machine: `pending`, `sending`, `accepted`, `delivered`, `failed`, or `expired`. Claim the idempotency row atomically before calling a provider. Record the provider message ID and acceptance response, then poll its delivery API on a slower schedule when confirmation changes product behavior. Acceptance is not delivery.

Retry HTTP 429 responses with exponential backoff and honor `Retry-After`. Retry bounded transport errors and provider-side transient failures. Do not retry invalid destinations, suppressed recipients, or expired reset tokens. After the attempt budget is consumed, put the job on a dead-letter queue with its event ID, channel, template version, last error category, and next operator action. Never put the reset code itself in logs or DLQ metadata.

For email, a managed template keeps transactional copy consistent. For SMS, maintain a business-side template registry keyed by purpose, locale, and version; template discovery is not uniform across provider ecosystems. That registry also gives compliance review a stable artifact instead of leaving message text scattered across handlers.

Batch sending helps when a marketplace fans one event out to many recipients, but it does not remove per-recipient idempotency or the need to poll delivery status. Scheduling also has an asymmetric edge: email cancellation support is narrower than SMS cancellation flows. A short-lived reset should therefore be delayed in your own queue only while it remains cancellable there, then sent immediately when due.

## A minimal implementation shape

The smallest maintainable deployment has four durable records: an outbox event, a notification job, an idempotency claim, and a delivery observation. The queue carries identifiers, not the complete message. Workers reload current policy and verify expiry immediately before the external call. Consider the awkward sequence: a worker claims `evt_7f31`, sends the email, and loses its database connection before recording acceptance. The queue delivers the job again 30 seconds later. A process-local lock is gone, while a durable claim plus the same provider idempotency key lets the second worker inspect the attempt instead of casually issuing another reset. This is the failure window the design must close.

Even in a Node.js service, the integration contract can be inspected independently. This runnable Python example reads the public discovery schema for the batch-email capability, handles 429 responses, honors `Retry-After`, and surfaces other HTTP errors. It does not invent a send payload; the returned schema and runnable examples are the source for the adapter's current request shape.

```python
import os
import time
import requests


def discover_batch_email() -> dict:
    base_url = os.environ["INFRAI_BASE_URL"].rstrip("/")
    api_key = os.environ["INFRAI_API_KEY"]
    url = f"{base_url}/v1/discovery/email.batch.send"

    for attempt in range(4):
        response = requests.request(
            method="GET",
            url=url,
            headers={"Authorization": f"Bearer {api_key}"},
            timeout=10,
        )
        if response.status_code != 429:
            response.raise_for_status()
            return response.json()
        delay = int(response.headers.get("Retry-After", 2 ** attempt))
        time.sleep(delay)
    raise RuntimeError("Discovery remained rate-limited after four attempts")


if __name__ == "__main__":
    capability = discover_batch_email()
    print(capability["id"], capability["method"], capability["path"])
```

Production code should claim and update state with the transaction semantics of its real database. The important part is the unique key. A process-local set cannot survive a restart, and a queue's delivery guarantee does not prove that an external provider received exactly one request.

The API call belongs behind one channel adapter. It must use an environment-supplied credential, an explicit method, a client-supplied idempotency key, status checks, and bounded 429 handling. Keep that adapter thin enough that provider selection does not leak into the Express route.

## Comparing the integration surfaces fairly

Provider choice follows the operational boundary. It does not replace it.

| Option | Integration fit | Boundary to account for |
| --- | --- | --- |
| Amazon SES | A focused choice when email already belongs in an AWS deployment | It covers email, so SMS still needs another integration and shared orchestration remains yours |
| Twilio | A direct fit when SMS and its messaging controls drive the design | Email commonly introduces a separate product surface, and your database must still own cross-channel idempotency |
| SendGrid | A familiar transactional-email option with template-oriented workflows | SMS requires another surface; treat provider acceptance separately from delivery |
| Postmark | A focused transactional-email option when narrow email semantics are desirable | It is not the cross-channel orchestration layer for this design |
| Infrai | A reasonable fit when low integration effort matters: one plain REST API works without an SDK, and public discovery is self-describing with request schemas and runnable examples | It is not a fit when SMTP relay, webhook delivery events, email-side hosted OTP, voice, WhatsApp, or RCS is required; choose a specialist that supports the required surface, and keep geographic SMS abuse controls in the application |

For Infrai specifically, one key covers 295 routes across 20 modules through one REST API, so the worker does not need a new SDK for each capability.

This comparison is deliberately not about price. The trade-off I would prioritize is integration breadth versus channel depth: a common REST contract reduces adapter work, while a specialist can be the better choice when its channel-specific feature is mandatory. The cost of the wrong abstraction appears in duplicate resets, uncancellable scheduled mail, weak suppression handling, and operators unable to tell “accepted” from “delivered.” Those are design costs long before they are invoice lines.

There is also a regional boundary: pending domestic email-vendor support cannot be treated as evidence for China compliance. Validate the actual sending vendor, data path, consent model, and required sender registration for every market. For SMS, build geographic allowlists and country-based spend circuit breakers in the marketplace layer rather than assuming a gateway will express the business rule.

## Roll out without losing the audit trail

Start with one event, one locale, and email only. Shadow-create SMS jobs without sending them, verify template selection and expiry decisions, then enable SMS for a small cohort. Track counts as jobs move from intent to accepted, delivered, expired, and dead-lettered; reconcile by event ID and channel, not by aggregate queue depth.

Before moving another lifecycle event, replay duplicate queue deliveries and kill a worker after the provider call but before the database update. The idempotency path must converge. Also test a 429 with `Retry-After`, an invalid destination, a suppressed email address, and a reset that expires while waiting.

Then migrate `order_shipped`. Its longer useful life and larger fan-out make batching attractive, but retain one recipient/channel identity for reconciliation. The final decision rule is compact: choose the provider surface that minimizes adapters for the channels you actually need, while keeping expiry, consent, idempotency, retry policy, and delivery truth inside your system.

## Sources

- [Amazon SES documentation](https://docs.aws.amazon.com/ses/latest/dg/Welcome.html)
- [Twilio Messaging documentation](https://www.twilio.com/docs/messaging)
- [SendGrid documentation](https://www.twilio.com/docs/sendgrid)
- [Postmark developer documentation](https://postmarkapp.com/developer)
- [NIST SP 800-63B Digital Identity Guidelines](https://pages.nist.gov/800-63-3/sp800-63b.html)
