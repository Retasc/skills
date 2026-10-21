# Dispatch: how next_issue decides

Part of the Retasc skill. [SKILL.md](../SKILL.md) is the overview; section numbers (§N) are shared across these files.

## 4. Dispatch: how `next_issue` decides

Eligibility, in the order the server checks it:

1. `work === true` (containers are never dispatched, never counted).
2. **Not quarantined.** A connector-created issue (`externalOrigin` set) with no `triagedAt` is
   counted as `awaitingTriage` and skipped. Only a human can sign it, in the Dash or with
   `retasc triage`. An agent cannot approve it.
3. Status is dispatchable (`todo`, `review`, or an **unheld** `doing`) and `claimedBy` is
   empty.
4. For a `review` issue: the caller's principal is not the submitter (author ≠ reviewer).
5. No **open blockers**: no `blocks` edge from a non-terminal issue in this project.
6. **Lane**: by default only issues assigned to the caller's own principal, or unassigned.
   Work in another human's lane is counted as `otherLanes`, not handed out. `allLanes:true`
   widens the pull; that decision is the human's, never yours.

Ordering of the eligible set:

- **Effective priority** = the strongest of the issue's own rank and every non-terminal issue
  it transitively `blocks`. A Low blocker of an Urgent issue inherits Urgent. Deadline floors
  (§3) apply to own rank first, so they inherit too.
- **Finish before start.** A `review` — finished work awaiting acceptance — is floored to at
  least **High** from the moment it enters review, and at EQUAL effective rank it sorts
  **ahead of any `todo`**. So what still goes before a fresh review is a `todo` whose
  effective rank reaches **Urgent** — marked Urgent, or floored there by its own deadline
  (§3: past 90% of its window, or overdue). Mark a real incident Urgent and it still
  dispatches first; nothing at a weaker rank jumps ahead of work that is already done. The
  floor then climbs with the review's own age — past ~21.6h it sorts Urgent, past 24h above
  everything — so after a day it passes those too. Like a deadline floor it inherits up the
  `blocks` chain, so a todo gating a review is pulled up with it. A **quarantined** external
  review (stranger-authored, not yet approved) is withheld from dispatch entirely and its
  clock starts only at approval, so an outside filer cannot climb by being ignored.
- Tiebreak: oldest `createdAt` first.

The response: `issue` (with `body`), `claimToken`, `claimExpiresAt`, `branch`, `activeClaims`
(other live leases in the project), `ready`, and when more than one issue was eligible an
advisory `wave` of up to 5 other ready issues (peek-only, NOT reserved). If the pick carries
a prior checkpoint: `resumed:true`, `checkpoint`, `checkpointBy`, `reclaimCount`. Read the
checkpoint before the body; it is where the last agent stopped.

Empty pull: `{issue:null, ready:0, otherLanes, blocked, claimed, awaitingTriage, tellHuman}`.
**Never report "the queue is empty" while holding a `tellHuman`.** Pass it on.

`next_batch(n?, claim?, allLanes?)`: the top N eligible issues (default 5, max 25), mutually
independent by construction. Default is a **peek** (claims nothing, metered as a read).
`claim:true` atomically claims all N, each with its own `claimToken` and `branch`. Claim a
batch only if you will run them concurrently; idle leases starve the fleet.

`claim_issue(identifier)`: a specific issue. Refused with a stated reason: `NOT_FOUND` (also a
sibling project's issue), `NOT_WORK`, `AWAITING_TRIAGE`, `NOT_ELIGIBLE: <status>`,
`AUTHOR_CANNOT_REVIEW`, `BLOCKED: … by X, Y`, `CLAIM_CAP`, `MONTHLY_CAP_REACHED`. A live lease
held by someone else is not takeable, except by its own token (§5).

Caps: **7 live claims per session** (`CLAIM_CAP`), checked before the queue. A monthly spend
cap per org.

