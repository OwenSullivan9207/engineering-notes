# Order Receipt Transactional Email Service: SPF/DKIM Domain Setup, No SMTP Relay

Short answer: choose a transactional email service whose API makes domain readiness, suppression decisions, accepted sends, and later delivery outcomes easy to represent in your own system; for an order receipt sent after payment settles, that integration clarity matters more than a long feature list.

This is an architecture decision record for a customer-support system, not a provider ranking. The trigger is a settled payment. The required result is a receipt that support can trace without treating an API acceptance, an inbox placement, and a human read as the same event. The design uses an API-first boundary and does not require an SMTP relay.

The catch is real: this choice is not suitable when an existing application can emit only SMTP, or when the organization already operates a governed relay with adequate authentication, suppression, and delivery telemetry. Keep that relay in those cases. Integration effort includes migration and operations, not just the number of lines in a quick-start example.

## Decision, invariants, and failure boundaries

Adopt a narrow mail adapter behind an application-owned outbox. The payment service records `payment.settled`; the receipt worker claims that durable work, checks policy, calls the provider adapter, and records the provider's acceptance identifier. A separate reconciliation process consumes or retrieves delivery outcomes. Customer support reads the application timeline, not a screenshot from a provider dashboard.

Four invariants define the choice. Production mail must use an approved sending domain with valid authentication records. The same order must not produce duplicate receipts when a worker retries. A recipient on the applicable suppression list must not be sent another receipt blindly. Finally, every attempt must retain an internal correlation ID that joins the order, the send request, and the delivery outcome.

Accepted is not delivered.

That distinction is the main failure boundary. A successful API response establishes that the remote service accepted a request; it does not establish inbox placement or reading. Open tracking is especially weak evidence because Apple Mail Privacy Protection can prevent senders from learning whether a recipient opened a message. For support work, record accepted, delivered, bounced, suppressed, and unknown as separate states. Don't turn an absent open into a failed receipt or an apparent open into proof that the customer read it.

The policy boundary matters too. An order receipt is transactional, but adding promotional copy can create a separate consent question. If marketing consent is requested, GDPR Article 7 requires the controller to be able to demonstrate consent and requires the request to be distinguishable from other matters in a clear, accessible form. Keep the receipt path and the marketing-subscription decision separate in data and code.

## What should an API-first transactional email service verify for receipt deliverability?

Start with control of the sending domain and the DNS records used for authentication. The evaluation must show the exact records to publish, expose a machine-readable verification state, and support a repeatable recheck after DNS changes. Include SPF and DKIM in the readiness review, but don't assume a generic “verified” badge proves every record and alignment condition your mail policy requires. Inspect the records, document their owner, and repeat the check during key rotation or domain changes.

Then walk the entire operational path with a fixed receipt fixture: a synthetic order ID, a non-production recipient, a currency, a total, and a settlement timestamp. The fixture should exercise domain readiness, suppression policy, API authentication, request validation, acceptance correlation, delivery reconciliation, and bounce handling. A polished template editor proves none of those boundaries.

I've seen delivery gaps become support problems because the integration stored one boolean called `sent`. That field collapses several facts that occur at different times. Consider order `ord_72A19`: payment settles at `14:03:11`, the outbox worker claims the job, and the first request is rate-limited with HTTP 429. The worker honors the service's retry guidance, releases the job until the next attempt, and does not hold the payment request open. When a later request is accepted, the adapter stores the external identifier alongside `ord_72A19`; reconciliation can then attach a delivery or bounce outcome to the same timeline. Until an outcome arrives, support sees `accepted`, not `delivered`. Your mileage may vary on the appropriate retry budget and reconciliation deadline, because those values depend on the support promise and provider contract, but the state distinctions should not vary.

Rate limits count.

During an implementation spike, test malformed addresses, a known suppressed test recipient, repeated processing of the same outbox item, credential rotation, DNS re-verification, a 429 response, and an outcome that arrives after the support freshness target. I'm not sure any documentation review alone can settle the integration-effort question. A small proof with recorded requests, responses, and operator steps will reveal how much glue the team actually owns.

## Options compared by integration work

This comparison uses architecture shapes rather than brands. Vendor contracts change; the engineering obligations remain legible.

| Option | Integration advantage | Cost and boundary |
|---|---|---|
| Direct transactional API plus application outbox | Explicit request validation and identifiers fit an application-owned audit trail; no SMTP relay is required | The team owns an adapter, idempotency policy, retry scheduling, and outcome reconciliation |
| Existing governed SMTP relay | Minimal application change when SMTP is already the standard transport | Envelope acceptance can be easy to confuse with delivery unless event data and suppression policy are integrated separately |
| Internal mail gateway in front of one or more services | Centralizes policy, credentials, and provider changes for many applications | Adds a service to operate and is excessive for one small receipt flow |
| Queue directly connected to a provider-specific worker | Removes polling from the payment request and can reduce custom dispatch code | Couples queue semantics and deployment tooling to the chosen integration |

The first option is the decision here because the customer-support timeline and the payment system already need explicit state transitions. It is not a universal recommendation. Stick with the governed SMTP relay when replacing it would add an adapter while leaving authentication, suppression, and event ownership unchanged. Choose an internal gateway when several teams need one enforced sending policy and can staff that gateway. A direct API is a poor fit if its delivery outcomes cannot be joined reliably to the application's correlation ID.

Price is deliberately absent from the primary decision. A low send rate does not compensate for manual DNS checks, unclear retry semantics, or support staff who must search another dashboard for each order. Compare expected engineering ownership and operating steps first; apply current, verified pricing to the resulting shortlist later.

## Critical path in Python

The code below keeps provider details outside the business workflow. It shows the states and duplicate-send guard that should survive a provider change. `MailTransport` is implemented by the selected service adapter after its current API contract has been verified; no guessed endpoint or response schema leaks into the architecture record.

```python
from dataclasses import dataclass
from datetime import datetime
from decimal import Decimal
from typing import Protocol


@dataclass(frozen=True)
class Receipt:
    order_id: str
    recipient: str
    total: Decimal
    currency: str
    settled_at: datetime


@dataclass(frozen=True)
class Acceptance:
    external_id: str
    accepted_at: datetime


class MailTransport(Protocol):
    def domain_ready(self) -> bool: ...

    def is_suppressed(self, recipient: str) -> bool: ...

    def send_receipt(self, receipt: Receipt, idempotency_key: str) -> Acceptance: ...


class ReceiptStore(Protocol):
    def acceptance_for(self, order_id: str) -> Acceptance | None: ...

    def record_suppressed(self, order_id: str, recipient: str) -> None: ...

    def record_acceptance(self, order_id: str, acceptance: Acceptance) -> None: ...


def dispatch_receipt(
    receipt: Receipt,
    transport: MailTransport,
    store: ReceiptStore,
) -> Acceptance | None:
    prior = store.acceptance_for(receipt.order_id)
    if prior is not None:
        return prior

    if not transport.domain_ready():
        raise RuntimeError("Sending domain is not ready")

    if transport.is_suppressed(receipt.recipient):
        store.record_suppressed(receipt.order_id, receipt.recipient)
        return None

    acceptance = transport.send_receipt(
        receipt,
        idempotency_key=f"order-receipt:{receipt.order_id}",
    )
    store.record_acceptance(receipt.order_id, acceptance)
    return acceptance
```

The store needs a uniqueness constraint on the logical receipt identity, not only the early lookup shown here, because two workers can race. The adapter must classify authentication and validation failures separately from rate limiting, apply only retries allowed by the verified contract, and preserve response details needed for diagnosis without logging message content or credentials. Reconciliation should update the timeline monotonically and retain the raw event identifier so repeated delivery events are harmless.

Deployment has its own gate. Verify the sending domain in the target environment, send the synthetic fixture, find its acceptance in the application log, reconcile its outcome, and make sure support can locate the record using the order ID. Rotate credentials through the approved secret-management process and confirm that logs redact them. The test should fail closed before production traffic if domain readiness is false.

Observability should answer three questions without a vendor dashboard: how many receipt jobs remain unclaimed, how long accepted messages have lacked a later outcome, and which terminal outcomes are increasing. Alerting on every transient retry creates noise; alerting on age and exhausted policy exposes customer impact. Keep recipient addresses and receipt content out of metric labels.

## Rejected design and its valid use case

The rejected design sends mail synchronously inside the payment-settlement request. It looks like less integration work because there is no outbox or worker. In practice, it couples payment latency to a remote mail call, makes retry ownership ambiguous, and invites duplicate sends when the caller cannot tell whether a timed-out request was accepted.

It can still be valid for a low-risk internal notification where loss or duplication is acceptable and no customer-facing request waits on the call. That isn't an order receipt. For the receipt path, durable intent plus asynchronous dispatch gives support a record even while delivery status is unknown.

Approval should therefore depend on evidence from the spike: DNS ownership and SPF/DKIM setup are documented, domain verification is observable, duplicate work produces one logical receipt, suppression behavior is explicit, 429 handling follows the verified contract, and delivery outcomes join back to the order. Reject the candidate if the integration leaves any of those as a dashboard-only assumption. The shortest SDK sample is beside the point; the smallest complete operational boundary wins.

## References

- https://support.apple.com/guide/iphone/use-mail-privacy-protection-iphf084865c7/ios
- https://gdpr-info.eu/art-7-gdpr/
