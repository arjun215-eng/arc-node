# Arc v0.7 follower-sync wedge — analysis and mitigation notes

*2026-07-22 · relates to [circlefin/arc-node#214](https://github.com/circlefin/arc-node/issues/214) ·
node: ip-172-31-31-151, follower on Arc Testnet (chain 5042002)*

Status: **mitigated locally** (single-height sync patch + 1-minute watchdog); **real fix ships in Arc v0.8**
(Circle confirmed on #214 that it's a known bug in the Malachite version v0.7 uses, already fixed upstream).
This document exists so the reasoning survives past the incident; it can be deleted after v0.8 lands and
the temp fork is retired.

## Incident record

| Date (UTC) | Wedged height | sync_height | Gap | Detected / recovered |
| --- | --- | --- | --- | --- |
| 2026-07-21 | 52,893,874 | — | — | ~18h stall (pre-watchdog); snapshot restore |
| 2026-07-22 13:32 | 53,088,995 | 53,089,002 | 7 | watchdog (5-min interval), CL restart |
| 2026-07-22 23:09 | 53,157,165 | 53,157,172 | 7 | watchdog, CL restart; interval tightened to 1 min after |

Terminal signature, identical each time: `tip_height` frozen, `sync_height` a few heights past it, all 5
parallel request slots occupied, log spamming
`Maximum number of parallel requests reached, skipping request for values max_parallel_requests=5 pending_requests=5`
~2x/second while the service stays "active". systemd catches crashes, not this.

## The known trigger (#214) and our patch

A multi-height range fetch that partially fails can skip a height: `sync_height` advances past the hole,
the 5 slots fill with post-gap heights whose blocks can never be committed (consensus applies in order),
the slots never free (nothing retires them but a commit), and the one request that would heal things —
the missing height — can never be issued because all slots are taken. Restarting the CL is the only cure.

**Local mitigation (deployed 2026-07-22):** `BATCH_SIZE: usize = 10 → 1` in
`crates/malachite-app/src/hardcoded_config.rs:158` — every sync request covers exactly one height, so a
fetch either delivers its height or fails whole and is re-requested; the partial-range split that creates
the gap cannot occur. Built from temp fork **arjun215-eng/arc-node @ `v0.7.3-single-height-sync`**; the
auto-upgrade will intentionally clobber it when v0.8 releases (EL binary untouched, so no version drift
fires early). Cost: ~10x more HTTP round trips during catch-up (2 per block instead of 2 per 10) —
acceptable for a follower; catch-up throughput remains well above the chain's ~2 blocks/s.

## Could there be a second trigger? (residual risk under batch_size=1)

The unifying shape of any second trigger is: **a pending-request entry that outlives any possibility of
its values being processed.** Once that exists, the wedge is mechanical and identical to #214 — the dead
entry pins a slot, consensus can't advance through the height it covers, and there's a nasty amplifier in
`set_sync_height`: it enforces the invariant that `sync_height` must never sit *inside* a pending range,
so even the recovery paths that try to reset `sync_height` back toward a hole get bumped past it — the
missing height is shielded by its own dead request. Reading the v0.7 code (Malachite rev `5cd137f`, crates
`sync/src/handle.rs` and `engine/src/sync.rs`), here are the concrete ways such a dead entry could form,
none of which involve range splitting:

1. **Successful fetch, dropped value — the buffer paths.** The likeliest class. When a response arrives
   for a height ahead of consensus, the engine buffers it in a bounded `sync_queue`. Two ways the value
   dies *after* the fetch succeeded: the queue is full (`"Failed to buffer sync response, queue is full"`
   — the value is dropped with just a warning) or the queue gets wiped by the height-restart path
   (`StartedHeight(restart)` → "Clear the sync queue"). In both cases the request already *completed*:
   the timer was canceled, the peer was credited, and the sync crate keeps the pending entry "as-is"
   expecting `prune_pending_requests` to clean it up "once consensus advances past this range" — which it
   never will, because the value it needed is gone and nothing re-requests it. `batch_size=1` doesn't help
   at all here; the failure happens after a perfect fetch. (Empirical check: grepped our entire log
   history including rotated archives — zero `queue is full` events ever, and queue-wiping height restarts
   are a validator-flavored path a pure follower shouldn't hit.)

2. **Processing-error attribution misses.** If a fetched value fails during processing (bad decode, cert
   rejection), the recovery path is `penalize_peer_and_retry`, which looks up the pending request by
   height. If that lookup misses — entry already removed, pruned, or reshaped by a race — it logs
   `"Received height for unknown request"` and does *nothing*: the height is never re-fetched. Any race
   between pruning and error delivery lands here.

3. **Timeout/response races at the actor boundary.** The engine actor and the sync crate keep separate
   books (`inflight` vs `pending_requests`). A response arriving just after a timeout is discarded as
   "unknown request" — fine, because the timeout already triggered a re-request. But the re-request path
   has a documented subtlety: if the send fails, the exclusion set is deliberately dropped and recovery
   leans on the peer scorer. A bug anywhere in that handoff — timeout fires, re-request send fails, state
   half-updated — leaves the same dead entry.

4. **Tip accounting on rollback.** `on_decided`/rollback paths move `tip_height` while old pending ranges
   survive. A follower shouldn't roll back, so this is the least likely class for this node.

Two takeaways. First, all three observed wedges had the small `sync_height − tip` gap (7 heights)
consistent with the range-split trigger, so the patch addresses the trigger we've actually seen — but
from logs alone the terminal signature of all four classes above is *identical* (slots pinned, tip
frozen), which is why the watchdog stays essential regardless. Second, this is exactly why the "gap
detection" proposal in #214 (`tip_height + 1 < min(pending_heights)` → force a re-request) was worth
advocating as a safety net: it's the only fix that heals every one of these classes, including ones
nobody has diagnosed yet.

## Why we did NOT implement gap detection in the fork

Considered and rejected for this box; every property that made the batch_size patch attractive flips:

- **Second repo to fork.** `BATCH_SIZE` is an arc-node constant; gap detection hooks the sync state
  machine (`pending_requests`, `tip_height`, re-request machinery), which lives in the Malachite sync
  crate — a git dependency on `circlefin/malachite`. We'd fork and maintain a second repo, repoint
  `Cargo.toml`, and regenerate the `Cargo.lock` the `--locked` build pins.
- **Subtler than the predicate suggests.** Legitimate transient states look gap-shaped (e.g. the
  "all eligible peers exhausted" path removes a pending entry and resets `sync_height` for retry next
  cycle). A naive checker double-requests heights already being handled — against three rate-limited
  public RPC endpoints, a misfiring checker is its own incident. It needs hysteresis, in-flight dedup,
  and interaction tests with `set_sync_height`'s bump-past-pending invariant.
- **Unvalidatable here.** With batch_size=1 the known trigger is gone, so the detector would sit dormant
  waiting for triggers we've never observed. Unreviewed consensus code, untestable in anger, running in
  production — the risk it adds plausibly exceeds the risk it removes.
- **Small, shrinking payoff.** It would turn a residual 1–2 minute watchdog-recovered stall (from a
  trigger class never seen on this node) into seconds of self-healing, on a testnet follower, until v0.8.

The right home for gap detection is upstream, reviewed and tested by the Malachite maintainers.
**Open question for #214:** does the v0.8 fix include the gap-detection safety net, or only the
range re-request? The answer determines whether this patch class is ever worth anyone's time.

## Defense-in-depth, current state

1. `batch_size=1` fork — removes the only observed trigger at the source.
2. Watchdog, 1-min interval, 5-min post-restart grace — caps *any* wedge class at ~1–2 min of stall
   (`/usr/local/bin/arc-node-watchdog`; escalates to EL+CL every 3rd counted stall).
3. Auto-upgrade — picks up v0.8 (the real fix) within a day of release and retires the fork by design.

## Code references (Malachite rev 5cd137f, as pinned by arc-node v0.7.3 Cargo.lock)

- `sync/src/handle.rs` — `on_valid_value_response` (partial-range split + remainder re-request),
  `on_invalid_value_response` / `re_request_values_from_peer_except` (full-failure path, peer-exhausted
  `sync_height` reset), `on_sync_request_timed_out`, `penalize_peer_and_retry` ("unknown request" miss),
  `set_sync_height` (bump-past-pending invariant).
- `engine/src/sync.rs` — bounded `sync_queue` buffering ("queue is full" drop), `StartedHeight(restart)`
  queue clear, `inflight` vs `pending_requests` double bookkeeping, late-response discard.
- `arc-node/crates/malachite-app/src/hardcoded_config.rs:158` — `BATCH_SIZE` (patched to 1 in fork).
- `arc-node/crates/malachite-app/src/rpc_sync/` — RPC-mode network actor (fetch → `SyncResponse(None)`
  on failure), `client.rs` JSON-RPC batch fetches.
