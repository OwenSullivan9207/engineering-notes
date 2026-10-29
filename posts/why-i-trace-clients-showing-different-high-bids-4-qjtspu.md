# Why I Trace Clients Showing Different High Bids: 4 Ordering Metrics

The decisive trade-off is delivery guarantee at fan-out: a high bid needs an ordered, recoverable record, while a collaborative cursor can tolerate loss and replacement. **Short answer:** assign every accepted bid a server-side sequence, make clients detect gaps, recover from an authoritative snapshot or replay, and measure convergence rather than raw message volume. Keep cursor traffic outside that correctness path. If two property managers see different leaders during a live auction, four signals usually locate the fault: accepted sequence, applied sequence, gap age, and snapshot version.

A single realtime channel can make these events look equivalent. They are not. The highest bid is business state; a cursor is a hint about where another editor is looking. Treating both as disposable creates missed bids. Treating both as durable lets cursor bursts delay the state people actually need.

That split matters.

## How should I debug clients that show different high bids?

Arrival order is not commit order. A reconnecting client may receive a fresh event before an older replay, an asynchronous handler may finish out of order, or one fan-out branch may miss a message. A local rule such as `max(amount)` also fails to express the full decision: the server may reject a bid, and equal amounts still need a deterministic accepted order.

Put the ordering fact at the authority that accepts bids. Each accepted state transition gets a monotonically increasing auction sequence. A client applies sequence `n` only after `n - 1`; duplicates are harmless, and a jump is evidence of a gap rather than permission to guess. The server remains the authority for the current leader and snapshot version.

A gap is proof.

This distinction matters in a property-management auction editor because presence traffic is tempting to place beside the bid stream. WebRTC data channels can be configured with ordered delivery, and retransmission limits are optional settings. The WebRTC specification also describes an unordered mode. Those controls are transport behavior, not a substitute for an application-level commit sequence or recovery protocol.

## Four signals that narrow the search

Start with one correlation key for the auction and record these values at acceptance, fan-out, client application, and recovery. Do not log bidder contact details or authentication material; identifiers should be scoped and suitable for the retention policy.

| Signal | Record | What a mismatch suggests |
| --- | --- | --- |
| Accepted sequence | Last sequence committed by the authority | The write path did not accept what the UI assumed it accepted |
| Applied sequence | Last contiguous sequence reduced by each client | Fan-out, reconnect, buffering, or client-handler trouble |
| Gap age | Time since the first missing sequence was observed | Recovery is stalled, not merely momentarily reordered |
| Snapshot version | Sequence represented by the fetched snapshot | A stale cache or an inconsistent snapshot boundary |

The useful comparison is `accepted_sequence - applied_sequence` for each connection, grouped by auction and client session. A zero gap with different displayed leaders points toward reducer or rendering logic. A positive gap on one client points toward delivery or recovery. Many clients falling behind at the same accepted sequence shifts attention toward the fan-out path. One client repeatedly taking snapshots shifts attention toward its connection lifecycle or apply loop. Now compare timestamps around the first missing sequence: acceptance time says the write became authoritative, fan-out time says a branch attempted delivery, apply time says the client actually reduced it, and recovery time says continuity returned. Missing one of those stages leaves a blind spot. The goal is not a crowded dashboard; it is a timeline that can distinguish an unaccepted bid from a committed bid trapped before display.

Count gaps separately from duplicate deliveries. A duplicate can be expected after retry and should be ignored by sequence; a gap threatens convergence. Also record recovery duration and the result: replay completed, snapshot replaced state, or session closed. Those labels are far more actionable than a generic reconnect counter.

## Make recovery explicit in the client

The reducer below refuses to leap over missing state. It buffers later messages, requests recovery once, and verifies that a snapshot covers the missing boundary. Cursor updates take a separate path and never advance the bid sequence.

```python
from dataclasses import dataclass, field
from typing import Callable


@dataclass
class AuctionView:
    auction_id: str
    applied_sequence: int = 0
    high_bid_cents: int = 0
    leader_id: str | None = None
    pending: dict[int, dict] = field(default_factory=dict)
    recovering: bool = False

    def receive_bid(self, event: dict, request_recovery: Callable[[str, int], None]) -> None:
        sequence = int(event["sequence"])
        if sequence <= self.applied_sequence:
            return

        self.pending[sequence] = event
        self._drain_contiguous()

        expected = self.applied_sequence + 1
        if self.pending and min(self.pending) > expected and not self.recovering:
            self.recovering = True
            request_recovery(self.auction_id, expected)

    def install_snapshot(self, snapshot: dict) -> None:
        version = int(snapshot["sequence"])
        if version < self.applied_sequence:
            return
        self.applied_sequence = version
        self.high_bid_cents = int(snapshot["high_bid_cents"])
        self.leader_id = snapshot["leader_id"]
        self.pending = {n: event for n, event in self.pending.items() if n > version}
        self.recovering = False
        self._drain_contiguous()

    def _drain_contiguous(self) -> None:
        while self.applied_sequence + 1 in self.pending:
            sequence = self.applied_sequence + 1
            event = self.pending.pop(sequence)
            self.high_bid_cents = int(event["high_bid_cents"])
            self.leader_id = event["leader_id"]
            self.applied_sequence = sequence
```

This example deliberately does not compute a winner from arrival order. It installs the authority's accepted state. In production, validate the event schema, cap the pending buffer, set a recovery deadline, and fail closed to a visible "updating" state when continuity cannot be restored. Silently showing a plausible old amount is the dangerous outcome.

Cursors use a different reducer: keep only the newest update for each participant and expire it after a bounded period. Losing an intermediate cursor coordinate is fine because the next coordinate replaces it. That traffic can use unordered or limited-retransmission delivery when the chosen transport and application requirements support it; bids cannot inherit that policy merely because they share a screen.

## Test the edges, not just the happy path

A deterministic test harness should inject one fault at a time: drop sequence 41, deliver 43 before 42, duplicate 44, reconnect between snapshot fetch and subscription, and delay a reducer callback. Assert that no client applies 43 while 42 is absent and that all clients converge to the authority's snapshot version after recovery.

Then test the race at the snapshot boundary. Subscribe-first designs must buffer events while fetching the snapshot; snapshot-first designs need a replay starting after the snapshot version. Without an explicit boundary, an event can land between those operations and disappear from the client's history. The metric signature is distinctive: snapshot version looks valid, the connection stays healthy, yet applied sequence remains behind accepted sequence.

I would deploy sequence validation in observe-only mode first. Emit gap metrics without changing the UI, confirm that identifiers and cardinality are controlled, then enable buffering and recovery for a small traffic slice. The rollback unit should be the new reducer, not the event schema: once producers emit sequences, old clients can continue ignoring the added field during the transition.

Keep the alert tied to user-visible risk. A brief out-of-order arrival that drains immediately is diagnostic data; a growing gap age or persistent accepted-versus-applied delta is an incident signal. Delivery rate alone cannot tell those cases apart.

Delivery is not convergence.

## Rollout decision

**Use durable ordering and recovery for accepted bids; use replaceable, lossy semantics for collaborative cursors.** Ship when injected drops and reorderings converge, snapshot boundaries are covered, and dashboards can separate authority, fan-out, and client-apply failures. Pause the rollout if a client can display a leader while its sequence has a known gap.

That condition is specific enough to operate. It also keeps the architecture honest: transport settings can improve delivery behavior, but only the application can define which state is authoritative and prove that every client has reached it.

## Sources

- https://www.w3.org/TR/webrtc/
