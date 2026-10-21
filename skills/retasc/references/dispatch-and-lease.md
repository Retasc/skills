# Dispatch and the lease

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

## 5. The lease

- **TTL 30 minutes** from the claim. `heartbeat` and `checkpoint` each reset it to a full 30
  minutes from that call. Behind the proxy nothing else touches it.
- The **reclaimer** (a server cron) frees any lease past expiry: clears the claim, `doing` →
  `todo` (a `review` claim stays `review`), bumps `reclaimCount`, logs `reclaimed`. The
  checkpoint and comments stay.
- **Who renews:**
  - Stock setup (the `retasc mcp-proxy` watchdog in front of the MCP): the proxy heartbeats
    every 10 minutes for every claim **it saw this session make**. Silent multi-hour work is
    safe. Process death, machine sleep, and key rotation stop it. A `/clear` does not: the
    window keeps renewing the claims the earlier conversation took (see the notice below).
  - Direct HTTP MCP with no proxy, echoing the `Mcp-Session-Id` we hand you at `initialize`
    (a cloud session, a claude.ai connector, any compliant MCP client): **your own calls**
    renew, a read as much as a write (RTSC-858). ONE call renews EVERY claim that session
    took, not just the issue you named. A silence over 30 minutes still lapses them, so
    `checkpoint` before one. **A reconnect ends it**: your client gets a new session id, and
    the claims taken under the old one stop renewing while `youHold` still reads true.
    `check_claim` returns `renewedByYourCalls`; when it goes `false`, `claim_issue` with your
    `claimToken` moves the lease onto the new session and renewal resumes. It is `null` where
    your calls cannot renew anything (behind the proxy, or with no session id); read
    `lastRenewedAt` there. Whether to re-claim is `reclaimWouldMove`: re-claim only when it is
    `true` (RTSC-1196). A re-claim of an issue this same session already holds only renews it,
    at the heartbeat price, and keeps your token.
  - Direct HTTP MCP sending no session id (raw `curl`, CI with a hand-rolled client): **you**
    renew. `checkpoint` before any stretch where you will not call Retasc. The claim response
    warns you ("I cannot see a local Retasc runner").
  - A claim this proxy never saw (another terminal, or before a restart): nobody renews it
    until you re-claim it with its token (below).
- **Prove it**: `check_claim` returns `renewedByYourCalls` (is THIS session's own activity
  renewing this claim right now) and `lastRenewedAt` (call it twice across a long stretch; if
  it moved, something is renewing you, frozen at claim time means nothing is). `check_claim`
  is the one tool that never renews anything, so neither reading can be an artefact of asking.
- **`claimToken`** fences every write on a held issue. Lost it (context summary, crash)? Omit
  it: the same session is still recognized for `heartbeat`, `checkpoint`, `release_issue`.
  Exception: a claim taken over a bare shared workspace key with no session key always needs
  the token.
- **After a restart or resume** your client is a new session. `check_claim` says
  `other_session` (the holder is your former self) and `save_issue status:done` fails with
  `CLAIM_MISMATCH`. Do not give the work back. Call `claim_issue` with the **old**
  `claimToken`: the lease moves onto this session (`rebound:true`, fresh token, checkpoint
  untouched) and the proxy starts renewing it again. Only the token can do this; a key cannot,
  because every agent in a folder shares the workspace key.
- **"This window still holds RTSC-X from the previous conversation"**: a `/clear` or
  `/resume` started a new conversation in a window that still holds and renews that claim.
  In Claude Code (and Codex) it opens the conversation; in every other harness it arrives
  on your next Retasc tool result (RTSC-1155). Where the harness restarted the proxy (Pi),
  the new proxy has already taken the claim over for you. Act on it, never ignore it: if the human asked to continue RTSC-X, carry
  on (you hold it; no new claim). If they asked for other work, `release_issue` RTSC-X first
  (the same session is recognized without the token; if the release is refused, tell them),
  then claim the new work. If it is unclear, ask them whether to keep or release it.
- `CLAIM_LOST` on a heartbeat means the lease already expired. It cannot be revived; re-claim.
- `release_issue(identifier, claimToken?, note?)` returns it to the pool (`todo`, or `review`
  for a review claim). Pass `note` as a final handoff.

`check_claim` statuses: `held` · `unclaimed` · `expired` · `other_session` · `unknown_session`.
