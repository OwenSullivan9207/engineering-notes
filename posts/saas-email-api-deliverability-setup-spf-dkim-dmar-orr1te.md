# SaaS Email API Deliverability Setup: SPF, DKIM, DMARC, Bounce Suppression and Polling

**Short answer:** Select a transactional email API by proving domain authentication, suppression enforcement, recoverable delivery events, and US/EU data handling in a test account; an attractive send endpoint is irrelevant if any one of those controls is missing.

The hard constraint is evidence. A successful API response shows that a request was accepted, while a SaaS application still has to connect that request to authentication state, later delivery events, and a decision about the next send. HTTP instead of an SMTP relay makes the application boundary easier to inspect, but it doesn't remove the operational work. The provider is one part of the control plane, not the control plane itself.

## How should a Node.js SaaS test SPF, DKIM, DMARC, and bounce suppression?

Start with a disposable subdomain and a test tenant. The evaluation should cover the complete lifecycle: produce the required DNS records, expose verification state through an API or other automatable interface, handle signing-key rotation, and keep staging separate from production. DKIM matters here because it signs message content and selected headers so a verifier can detect modification and associate a signature with a domain. The exact signing and verification model is defined in RFC 6376; a provider dashboard screenshot isn't equivalent evidence.

SPF, DKIM, and DMARC should be treated as separate checks in the acceptance plan. Ask the candidate to show which domain each check uses, how alignment is reported, and what the application can observe while DNS changes are incomplete. For multi-tenant SaaS, add an isolation test: one tenant's domain configuration must not expose or alter another tenant's state. Registrar credentials do not belong in the mail application's database.

Don't move directly to an enforcing DMARC policy just to finish a setup ticket. Inventory every legitimate source that sends with the domain first, observe the reports, and tighten policy only after the evidence supports it. The uncertain part is receiver behavior and the existing mix of senders; a single inbox test can't resolve either one. Authentication should also be checked independently for password resets, invoices, support notifications, and any remaining marketing stream because they may not share the same sending path.

Suppression is an admission-control decision, not a reporting feature. Before a send enters the provider queue, the application should check whether the recipient is blocked by a hard bounce, complaint, unsubscribe, or an administrator action. A soft failure belongs to a bounded retry policy. An expired one-time code belongs nowhere: OWASP's forgot-password guidance says reset codes or tokens should expire after an appropriate period and be invalidated after use, so a delayed retry must never revive an old authentication attempt.

The acceptance test should be blunt. Submit a valid address, a malformed address, and a recipient already in the local suppression set. Confirm that the suppressed case never reaches the outbound adapter. Then inject duplicate and out-of-order delivery events and verify that the local state stays consistent. If suppression can only be inspected manually, or events can't be recovered after the live callback is missed, the service has failed the SaaS requirement even though its basic send demo works.

## Make the event ledger the system of record

Acceptance is not delivery.

Give each logical message an internal message ID and each delivery attempt a separate attempt ID. Store the remote message ID returned by the adapter, but never use it as the only join key. The event ledger should accept duplicates, preserve the original event time, reject invalid state regressions, and retain enough context to explain why another send was allowed or suppressed. This is especially important for short-lived authentication mail: a late event can still be useful for diagnosis even when the code itself is already invalid.

Webhooks provide a prompt event path. Event polling provides recovery when a callback was missed, rejected, or delayed during deployment. A sound integration uses both when both are available: verify the webhook, enqueue its raw event for asynchronous handling, and periodically poll from a durable cursor with a small overlap. Commit the next cursor only after the corresponding event page and state changes commit together. No drama, just bookkeeping.

The provider-specific details can stay behind a small Python boundary even when the calling service is Node.js. The point is the contract, not the language of the production process:

```python
from dataclasses import dataclass
from typing import Protocol


@dataclass(frozen=True)
class DeliveryEvent:
    event_id: str
    attempt_id: str
    kind: str
    occurred_at: str


class MailAdapter(Protocol):
    def submit(
        self,
        *,
        attempt_id: str,
        recipient: str,
        template_id: str,
        variables: dict[str, str],
    ) -> str: ...

    def read_events(
        self, *, cursor: str | None
    ) -> tuple[list[DeliveryEvent], str]: ...


def reconcile(adapter: MailAdapter, cursor: str | None, ledger) -> str:
    events, next_cursor = adapter.read_events(cursor=cursor)
    with ledger.transaction() as transaction:
        for event in events:
            if transaction.insert_event_if_absent(event.event_id, event):
                transaction.advance_attempt(
                    event.attempt_id, event.kind, event.occurred_at
                )
        transaction.save_cursor(next_cursor)
    return next_cursor
```

The same contract needs a rate-limit path. A submission worker should distinguish retryable transport pressure from a permanent recipient decision, apply bounded backoff, and keep the same logical identity across retries. It must also stop when an OTP expires or when suppression state changes while an attempt is waiting. Otherwise a retry queue quietly becomes a second, less visible sending system.

## Compare operational proof, not API ergonomics

Most evaluations spend too much time on request syntax. A short proof should instead force each candidate through the failures the application must survive. The table below is deliberately evidence-oriented; every row should produce an artifact that another engineer can inspect.

| Constraint | Evidence from the proof | Reject when |
| --- | --- | --- |
| Domain control | Per-domain verification state, rotation procedure, tenant isolation test | Authentication is shared across unrelated tenants |
| Suppression | Automatic enforcement plus an exportable reason and timestamp | Known bad recipients can re-enter the send path |
| Event recovery | Verified callbacks and replayable or pollable event history | A missed callback creates a permanent blind spot |
| HTTP integration | Stable message identifiers, scoped credentials, documented limits | Core transactional sends require an SMTP relay |
| US/EU data handling | Field-level data-flow map, retention terms, deletion procedure, support-access scope | A region label has no defined data boundary |
| Operations | Test isolation, credential rotation, audit history, observable queue age | Production credentials are needed for local tests |

Regional selection needs more than choosing an endpoint marked “EU.” Trace recipient addresses, message bodies, template variables, event payloads, logs, backups, and support access. Record where each category is processed, how long it is retained, who can retrieve it, and how deletion propagates. Legal counsel decides which obligations apply; engineering has to supply an accurate data-flow map and implement the resulting retention controls. Your mileage may vary with message content and customer contracts, which is precisely why a generic region badge can't settle the choice.

There are honest trade-offs. Polling-only event recovery can fit low-urgency notifications whose state may lag, but it is not suitable for short-lived OTP workflows that need prompt feedback. Callback-only ingestion can be adequate when retry history is durable and the receiver is continuously operated; choose recoverable event history when missed-callback repair is a hard requirement. An SMTP relay can remain appropriate for a legacy application that cannot issue authenticated HTTP requests. For a new service, an HTTP API usually exposes identifiers, credentials, payloads, and errors in a form that is easier to put behind a typed adapter, but the team still owns retries and observability.

Cost belongs in the final traffic model alongside engineering time, retention, support access, and expected event volume. It shouldn't compensate for weak isolation or an unrecoverable event stream. Likewise, a broad feature list cannot rescue a candidate that offers no defensible answer for suppression before send.

## Roll out one message stream at a time

Begin with a low-risk notification stream on its own subdomain. Keep password reset and OTP mail isolated until authentication, suppression, and reconciliation have been observed under production-like traffic. Route by tenant or another deterministic cohort so the same logical attempt never reaches two delivery systems, and define rollback before changing DNS or templates.

Before expanding the cohort, exercise malformed input, duplicate events, out-of-order events, expiring codes, a changed suppression decision, rate limiting, a receiver restart, and a stale polling cursor. Alert on queue age, cursor age, accepted attempts without later events, and changes in bounce or complaint outcomes. Support tools should distinguish “accepted,” “reported delivered,” and “seen by a person”; those states are not interchangeable.

Keep the first move small.

The durable decision record is the useful output of the selection process. It should name the constraints that won, the evidence collected, the rejected boundaries, the rollback trigger, and the conditions that force a new evaluation. Revisit it when traffic shape, tenant isolation, message sensitivity, or regional commitments change. That discipline does more for deliverability than polishing the first send call.

## References

- RFC 6376, DomainKeys Identified Mail (DKIM): https://datatracker.ietf.org/doc/html/rfc6376
- OWASP Forgot Password Cheat Sheet: https://cheatsheetseries.owasp.org/cheatsheets/Forgot_Password_Cheat_Sheet.html
