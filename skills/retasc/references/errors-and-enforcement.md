# Errors, and what Retasc enforces

Part of the Retasc skill. [SKILL.md](../SKILL.md) is the overview; section numbers (§N) are shared across these files.

## 14. Error vocabulary, and what to do

| Error | Meaning | Do |
|---|---|---|
| `UNAUTHORIZED` | Key unknown or revoked, member suspended, org deleted, or project scope violated. One message for all of those, deliberately. The one exception is `UNAUTHORIZED: session key expired`, a session key the server retired because it went unused or silent for a week, which the proxy replaces by itself (RTSC-1194, RTSC-1195). | §8 step 2. Never loop on `bind`. |
| `NOT_FOUND: X` | No such issue in **this project** (sibling projects read as absent). | Check the prefix; `list_projects`. |
| `NOT_WORK` | A container (`work:false`). | Read it; claim its children. |
| `AWAITING_TRIAGE` | External intake not yet signed. | Relay to the human: Dash or `retasc triage`. |
| `NOT_ELIGIBLE: <status>` | Terminal, or the `blocked` status. | Leave it. |
| `BLOCKED: … by A, B` | Open blockers. | Take the blockers (dispatch already ranks them first). |
| `CLAIM_CAP` | 7 live claims on this session. | Finish or release before claiming more. |
| `CLAIM_MISMATCH` | Another live session holds it (often your former self). | `claim_issue` with your old `claimToken`. |
| `CLAIM_LOST` | The lease expired. | Re-claim; expect a `resumed` checkpoint. |
| `CLAIM_REQUIRED` | Promoting a review without holding its claim. | Claim the review first. |
| `AUTHOR_CANNOT_REVIEW` | You submitted it. | A different principal accepts. |
| `HANDOFF_REQUIRED` | Entering review without evidence. | Resend with `handoff:"branch / PR / commit"`. |
| `INVALID: author ≠ reviewer`, reviewer inactive, placeholder | Bad `assignee` on a review. | Name an active, different human. |
| `INVALID: cancel_reason required` | | Add `cancelReason`. |
| `WORK_REQUIRED` / `DEPENDENCY_REQUIRED` | Create without `work` / without `blockedBy` or `noDependency`. | Add them. |
| `DUPLICATE_CHECK_REQUIRED` | Open issues look like the same or adjacent work, and you have not said you read them. | Read the ones it names. If one IS this work, don't create — comment or claim. Otherwise re-call with `reviewedSimilar:["RTSC-NN", …]`. |
| `CHANGE_TYPE_REQUIRED` | Create without `changeType`. | Add one of the eleven; the refusal lists them. |
| `INVALID: dependency cycle` | The `blocks` edge would loop. | Rethink the edge. |
| `BILLING_INACTIVE` / `MONTHLY_CAP_REACHED` | The org is gated. | Relay `tellHuman`; stop value-bearing writes. |
| `EXPIRED` / `CONSUMED` | An invite or token is dead or already used. | Ask for a fresh one. |

## 15. What Retasc enforces, and what it does not

**Enforced by the server** (a rule, not a request): one holder per issue (atomic claim); the
30-minute lease with reclaim; `claimToken` / session fencing on every write to a held issue
and on `done` / `canceled`; dependency blocking from `blocks` edges (never dispatched, never
claimable); effective-priority order; lane scoping by default; the 7-claim cap and the monthly
spend cap; project as a hard boundary; `work`, the dependency declaration and the duplicate
acknowledgement on create;
`cancelReason`; `handoff` plus an active, different-principal `assignee` to enter and stay in
`review`; holding the review claim to accept; author ≠ reviewer; quarantine of external intake
until a human signs it; owner / admin gates on membership, imports, connectors, billing.

**Not enforced, and not modelled**: branches, PRs, CI, commit messages, merges, tests,
"verified". `done` never waits on a check. Any process beyond the claim contract (review
policy, commit format, who merges) is the team's own convention. Do not invent a workflow and
attribute it to Retasc.

**The one exception: isolation.** Retasc's promise is that no two agents collide on the same
work. The server guarantees that for *issues* (claims are atomic) but cannot see *files*, so
the promise needs one condition on the client side: **one isolated working copy per agent,
per issue.** That is a workflow opinion, and Retasc holds it deliberately, because without it
the guarantee is false.

The implementation is **git worktrees**. Git is assumed for any work that edits files, and a
worktree is the strongest isolation git offers: shared objects and refs, instant creation,
native reaping, and the whole fleet visible in one `git worktree list`. A clone or a per-task
container gives the same isolation and is acceptable where a worktree is not possible; a
shared checkout is not.

This is the same category as `author ≠ reviewer`: a rule held because the guarantee depends
on it, not because it is a nicer process. It is REQUIRED by the claim contract and stated on
every claim; the server does not check it, because the server cannot see git. Because git and
worktrees are assumed, a client-side component may *read* local git to make in-flight work
visible. What the server may do with that fact is bounded the same way lane scoping is:
**dispatch may withhold on it** (`next_issue` / `next_batch` skip work someone is seen to be
on), but **nothing may refuse an explicit `claim_issue`** — the worker reclaims its own work,
and a deliberate override stays possible and attributable.

**How that is built (RTSC-962): the provisional hold.** Your mcp-proxy watches for git
worktrees on `rtsc-NN/<slug>` branches that exist on your machine and that no live claim
covers, and reports them with `report_worktrees` — a machine tool you never call yourself.
The server records each as a **hold** on that issue. A hold does three things and nothing
else: it appears on `get_issue` / `list_issues` as `hold {branch, detectedAt, lastSeenAt}`
and as a rider on the read; `next_issue` / `next_batch` withhold the issue and
`queue_status` counts it as `heldByObservation`; and `save_issue work:false` is refused
over it. It lapses 60 minutes after the last sighting, and an inactive worktree (clean,
no commit in 24h) is never reported at all, so an abandoned directory cannot park work.

**A hold is not a lease.** It has no token, `heartbeat` / `checkpoint` / `done` do not
accept one, and `check_claim` reports it as not held. **`claim_issue` is never refused
because of one** — a named claim from any session converts it into a real claim, which is
how an agent that forgot to claim gets its own work back. If a read tells you somebody is
on an issue: claim it if that is you, and if it is not, go and look at the branch rather
than starting a second one.

**What it does NOT cover.** This widens the promise from "no two agents *hold* the same
issue" to "no two agents *work* the same issue" — **for sessions running the mcp-proxy**.
An agent talking to `mcp.retasc.com` directly (a cloud session, a chat connector, a phone)
has no local process to see its filesystem, so for those the isolation rule stays advisory.
