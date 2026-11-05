# Marketplace MX Verification: 3 Node.js Gates for Mail Exchange and Forwarding

Fast marketplace onboarding creates a timing conflict: proving that a seller controls a domain should not force an immediate change to its incoming mail. Resolve that conflict by separating ownership proof, routing readiness, and activation into three gates. TL;DR: verify a fresh domain challenge first, check the receiving provider's intended MX records separately, and activate only when the mail exchange setup is ready. An API response accepting a DNS change isn't evidence that every sender will see it.

This note treats a Node.js onboarding API as the coordinator. The focused policy example uses Python; the policy applies independently of the API runtime. The design question is how quickly a seller can finish onboarding while existing mail continues to reach a receiver prepared to handle it.

## What do MX records prove about mail exchange setup?

An MX record describes a mail exchanger and its preference for a domain. Under SMTP, lower preference values are preferred; equal preferences don't establish a fixed first choice. These priorities answer where a sender should attempt delivery. They don't bind a marketplace account to the person who requested onboarding.

Separate the proofs.

For an illustrative seller domain, `seller.example`, issue a fresh challenge tied to the account, the exact domain, and an expiry. Require its publication at a designated DNS name before marking ownership verified. This is a proposed application protocol, not a claim that SMTP defines marketplace verification. Keep the challenge separate from the MX records so verification can proceed without changing the seller's production mail destination. A matching shared mail host is too weak an account-binding rule: its name identifies routing infrastructure, while the onboarding decision concerns which account may claim this particular domain.

Ownership also has a narrow meaning. It authorizes the marketplace's domain association under this policy; it doesn't establish consent to receive marketing or prove that an OTP can reach a particular mailbox.

## Derive activation from the observations you actually have

The second gate asks whether the observed MX set matches the intended receivers and preferences. Compare complete record sets, including the preference numbers. In illustrative configuration data, a primary at preference 10 and a secondary at 20 express an order of attempts, not a traffic split. Publishing both an old receiver and a new receiver at equal preference should therefore not be treated as a controlled canary rollout. SMTP's selection rules aren't an application deployment scheduler. A receiving provider's setup instructions supply the intended targets; the coordinator still needs to observe the resulting DNS records.

DNS caching puts a boundary around the observation. A recursive resolver can retain a record until its TTL expires, and negative answers can also be cached. Lowering a TTL doesn't retroactively shorten the lifetime of an answer already cached under an earlier TTL. One successful lookup cannot support the claim that all senders have switched. Distinguish the time a write was accepted from the time a resolver returned the intended answer, and preserve the observation time with the result.

The third gate concerns delivery through the new path. Before activation, require a controlled delivery observation appropriate to the feature being enabled. For a mailbox verification flow, that might mean completion of a fresh challenge delivered through the intended receiver. Treat it as evidence for that path at that time. It cannot guarantee delivery from every sender or future placement in an inbox.

Keep pending explicit.

A pending DNS observation should leave the seller in a resumable onboarding state. Don't turn each retry into another OTP email. The application can poll DNS with bounded retries and backoff while maintaining a separate cooldown for messages, which avoids tying DNS uncertainty to repeated sends. These are proposed controls: choose their intervals from the onboarding deadline and the delivery policy, rather than presenting an arbitrary polling interval as a DNS standard.

## Keep the API decision smaller than the DNS machinery

A coordinator needs a durable distinction between evidence and eligibility. The following Python example illustrates only the activation policy. Its booleans represent evidence already validated for the current account, exact domain, and current configuration revision; the function doesn't perform DNS queries, parse TXT answers, or authenticate incoming callbacks.

```python
from dataclasses import dataclass
from datetime import datetime


@dataclass(frozen=True)
class Evidence:
    ownership_matches: bool
    ownership_expires_at: datetime
    mx_matches_plan: bool
    delivery_confirmed: bool
    current_revision: int
    observed_revision: int


def activation_state(evidence: Evidence, now: datetime) -> str:
    if now.utcoffset() is None or evidence.ownership_expires_at.utcoffset() is None:
        raise ValueError("Timezone-aware timestamps are required")
    if not evidence.ownership_matches:
        return "awaiting_ownership"
    if now >= evidence.ownership_expires_at:
        return "renew_ownership_challenge"
    if evidence.observed_revision != evidence.current_revision:
        return "awaiting_current_evidence"
    if not evidence.mx_matches_plan:
        return "awaiting_routing"
    if not evidence.delivery_confirmed:
        return "awaiting_delivery"
    return "eligible_for_activation"
```

Eligibility is a decision input. Persist activation atomically against the same configuration revision so an edit between evaluation and commit cannot activate a different plan. A delayed successful check for the previous receiver must not approve the replacement receiver. Likewise, challenge expiry belongs in the transition check, rather than only in a background cleanup job.

For operations, record the observed RRset, resolver, timestamp, intended revision, and transition reason. Keep challenge values and mailbox details out of general logs; use identifiers and restrict access to verification evidence. Useful measurements include time spent awaiting ownership, time awaiting routing, and the number of messages sent per verification attempt. These distinguish a DNS wait from a delivery problem without encouraging the support team to keep resending codes. Polling consumes resolver and worker capacity too, so bounded retries are also an operational cost control.

## Compare receiving mail with forwarding it

Once the gates are clear, the routing choice becomes easier to evaluate. Direct reception and forwarding have different acceptance boundaries. An MX preference cannot express those boundaries for you.

| Choice | What the marketplace must establish | Cutover implication |
| --- | --- | --- |
| Direct receiver | The receiver accepts the domain and the required recipients | Prepare acceptance before publishing its MX records |
| Forwarding receiver | The forwarder accepts the domain and can deliver to the destination mailbox | Validate both the inbound acceptance and the onward delivery |
| Old and new receivers during transition | Both can handle mail arriving during the DNS cache overlap | Keep both paths operational while cached answers can remain in use |

Forwarding deserves particular attention in OTP flows. DMARC evaluates alignment with the visible From domain using authenticated identifiers from SPF or DKIM. A forwarding hop can disrupt SPF-based authentication; an aligned DKIM signature may still allow DMARC to pass if it survives transit. Message modifications can affect signature validation. RFC 7489 discusses these indirect-flow difficulties, so test the final destination's authentication result instead of assuming that successful acceptance by the forwarding host establishes successful authentication downstream.

DMARC success isn't an inbox-placement promise.

A higher-preference backup MX is also a receiving responsibility. If it is published, plan for its acceptance, queueing, and onward handling; don't use it as a placeholder for a receiver that has never been exercised. The trade-off is how much of the delivery path the team can observe and operate during transition. A limitation of forwarding is the additional authentication boundary. Direct reception places the receiving responsibility on the selected destination. Neither choice removes the DNS cache interval.

## Roll out with a reversible activation boundary

Start by deploying the pending states and revision checks before changing production MX records. Exercise expiry at the exact deadline, a stale successful check after a configuration edit, an equal-preference MX set, a cached negative response, and a destination that accepts the forwarded message but doesn't complete the mailbox challenge. Those tests target decisions the coordinator can get wrong.

Then prepare the receiving path, obtain ownership evidence, publish the intended routing change, and collect current delivery evidence before activation. Keep the previous receiver ready during the planned overlap. DNS rollback is another DNS change with caching implications, so it cannot promise an immediate return for every sender. If evidence becomes stale or incomplete, hold onboarding at the relevant gate and show its reason.

The release criterion is compact: the account has a valid domain claim, the current routing plan has been observed, and the intended delivery path has completed its check. That boundary lets onboarding advance as soon as its evidence supports it while preserving a truthful distinction between verification and mail cutover.

## Sources

References:

- RFC 5321, SMTP mail routing and MX preference handling: https://datatracker.ietf.org/doc/html/rfc5321#section-5
- RFC 1034, DNS concepts and caching: https://datatracker.ietf.org/doc/html/rfc1034
- RFC 2308, negative caching of DNS answers: https://datatracker.ietf.org/doc/html/rfc2308
- RFC 7489, DMARC alignment and indirect email flows: https://datatracker.ietf.org/doc/html/rfc7489
