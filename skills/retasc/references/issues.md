# Issues, identity and the issue lifecycle

Part of the Retasc skill. [SKILL.md](../SKILL.md) is the overview; section numbers (§N) are shared across these files.

## 1. The objects the server has

These are the tables. A noun that is not here does not exist in Retasc.

| Object | What it is | Load-bearing fields |
|---|---|---|
| **org** | The customer. Billing boundary and hard data-isolation boundary. Content is encrypted at rest under a per-org key; deleting the org destroys the key. | `name`, `slug`, `timezone` (unset = UTC), billing status |
| **project** | The unit agents pull from. Owns the identifier prefix and the counter. **A hard boundary**: a key bound to project A cannot read, claim, or link project B's issues; they read as `NOT_FOUND`. | `prefix` (e.g. `ACME`), `counter` |
| **issue** | One item of work, or one container. Identifier `{PREFIX}-{n}`, immutable, unique per org. | `title`, `body`, `priority`, `status`, `work`, `assigneeId`, `dueAt`/`dueDate`, claim fields, `checkpoint`, `reviewSubmittedBy`, `externalOrigin` + `triagedAt` |
| **member** | A human or an agent in an org; `isAgent` tells them apart. | human: `userId`, `role` (owner / admin / member), `projectIds` (unset = all). agent: `principalUserId` (the human it acts for), `runtime` |
| **api_key** | The whole of an agent's identity and routing: `orgId` + `projectId` + `memberId`, stored hashed. A **workspace key** is what `bind` mints for a folder. A **session key** (`parentKeyId` set) is what the watchdog proxy mints per conversation so parallel sessions of one agent are told apart. | `workspace` (folder leaf name, self-reported), `transcriptId`, `model`, `clientName` |
| **relation** | An edge between two issues. `blocks` / `blocked_by` drive dispatch; `related` and `duplicate` are informational. Blocking is **computed** from edges at read time, never stored on the issue. | `sourceId`, `targetId`, `type` |
| **comment** | Shared memory on an issue. Retract, never delete. | `body`, `authorId`, `retractedAt` + `retractionNote` |
| **attachment** | A link or an uploaded file on an issue. Obsolete, never delete. Files are size-priced. | `url`, `storageId`, `contentType`, `bytes` |
| **label** | Org-wide tag, human-side only. **Labels never dispatch.** | `name`, `color` |
| **invite** | A single-use bearer code that adds a human to an org as `member`. May be addressed to a known user so their agent can accept it. | `role`, `projectIds`, `inviteeUserId`, `expiresAt` |
| **setup token** | One-shot Dash-minted credential meaning "mint one agent key for this org + project". Redeemed by `retasc bind --setup <token>` with no sign-in. | `orgId`, `projectId`, `expiresAt` |
| **activity** | Append-only log per issue: created, claimed, reclaimed, status_changed, commented. | `verb`, `actorId`, `at` |

There is no parent field on the MCP create path. Structure is expressed with `blocks` edges
and `work:false` containers.

### What the server cannot see

- **No folder, path, repo, branch, PR, or CI.** MCP is stateless HTTP with a bearer token. The
  server never learns a path. A folder's config chooses *which key is sent*; that is the whole
  mechanism. `api_keys.workspace` is a leaf name the CLI volunteers, self-reported, unverified.
- **No git.** `done` never reads a branch. The `branch` in a claim response is a name the
  server *computes* (`{prefix-lowercase}-{n}/{slug}`) so every runtime agrees; nothing checks
  that it was used.
- **No session beyond the key.** The `initialize` handshake is the only moment client facts
  exist, which is why runtime and model labels live on the key row.

## 2. Identity and authority

- A key resolves to an **agent member**, which carries a **principal** (the human who minted
  it). The agent has no authority of its own; every permission check reads the principal's
  role and project scope. `whoami` returns `member.name`, `member.runtime`,
  `member.principal`, `member.session`, `org`, `project`, `pendingInvites`, `firstCall`.
- Roles: **owner** (one per org: delete, export, billing, everything), **admin** (runs the org
  day to day: invites, imports, connectors, offboarding ordinary members; cannot act on an
  owner or a peer admin), **member** (works). Members are free and unlimited; billing is per
  action, never per seat.
- **`assigneeId` is always a human.** It is a routing lane ("whose responsibility"), never who
  is executing. Claiming is a separate axis. Claiming an unassigned issue fills the assignee
  with the claimant's principal so the Dash can say "worked by".
- Three orthogonal axes on every issue: **author** (`createdBy`), **assignee** (human lane),
  **claim holder** (an agent session with a live lease).
- People are **suspended** (reversible). Agents are **retired** (terminal; a revoked key cannot
  be un-revoked).
- A human can be **scoped** to a subset of projects (`projectIds`). Their agents inherit it.

## 3. The issue lifecycle

### Status set (six, fixed)

`todo` · `doing` · `blocked` · `review` · `done` · `canceled`

- `doing` means **a lease was taken**; it is live until `claimExpiresAt`. Claim ⇒ `doing`.
  Release or reclaim ⇒ `todo`. Between expiry and the reclaimer's next sweep the issue is
  still `doing` but unheld, and dispatch treats it as free.
- `blocked` is **manual / external only** ("waiting on a vendor"). It is NOT how dependency
  blocking is expressed. Dependency blocking is computed from `blocks` edges and never shows
  in the status field. A `blocked` issue is unclaimable (`NOT_ELIGIBLE`). Never park an issue
  in `blocked` because a PR is waiting on your own human; let the lease lapse, the checkpoint
  carries the PR.
- `review` = work finished, pending human acceptance. **Non-terminal**: it still blocks its
  dependents. Opt-in per issue.
- `done` and `canceled` are **terminal**. Only these clear a dependent's blocker. `canceled`
  requires `cancelReason`.
- `closedAt` is set iff currently terminal (cleared on reopen); `completedAt` / `canceledAt`
  stick for history.

### Work vs container (`work`)

Decided once at creation, mandatory on the MCP create path. `work:true` = an agent finishes it
by changing files or systems. `work:false` = it only organizes other issues (epic, milestone,
plan) and is **never dispatched or claimable** (`NOT_WORK`); by convention it is `done` when
its children are. A container cannot be in `review`; resend `work:true` first, or close it.

### Kind of change (`changeType`)

Mandatory on the MCP create path, one value, from Conventional Commits and nothing else:

`feat` · `fix` · `docs` · `style` · `refactor` · `perf` · `test` · `build` · `ci` · `chore` · `revert`

Pick what the change **is**, not what it touches — area is what `labels` are for, and they are
unaffected and still many-per-issue. You are the one who knows: you have just written the body.
It is one value because spend is split by it and a stack has to add up, and it is correctable
with a later `save_issue` if the work turns out to be something else.

### Priority

`0 = None, 1 = Urgent, 2 = High, 3 = Medium, 4 = Low`. **0 sorts last.**

### Deadlines

**You are the only one who can set one.** `dueAt` is written over MCP and nowhere else —
the Dash has no date field, so a deadline a person said out loud and you did not record
does not exist. Two rules when you file work:

- **The person named a time** ("by Friday", "before the demo", "end of the month") →
  convert it to the instant they meant, in their timezone, and send `dueAt`.
- **The work is plainly time-bound and nobody said when** → ask, once, while filing. A
  deadline added days later is a deadline that never dispatched.

Most work carries no commitment and should carry no date: a guessed deadline is worse
than none, because dispatch believes it.

`dueAt` (epoch ms, precise) or `dueDate` (day-granular, IMPORT ONLY; resolves to end of
day in the org timezone). Every read derives `slaState`: `ok` (< 75% of the window
elapsed), `warning` (≥ 75%), `breaching` (≥ 90%, not yet past), `breached` (past).
Dispatch floors effective priority from it: ≥ 75% → at least High, ≥ 90% → at least
Urgent, breached → above Urgent. The floor inherits up the `blocks` chain like raw
priority.

### Creating an issue (`save_issue` without `identifier`)

Required: `title`, `work`, `changeType`, and **one** dependency declaration (`blockedBy: [ids]` or
`noDependency: "one-line reason"`) — plus `reviewedSimilar`, but only when open issues actually
look like this work. A create missing any of them is rejected (`WORK_REQUIRED`,
`CHANGE_TYPE_REQUIRED`, `DEPENDENCY_REQUIRED`, `DUPLICATE_CHECK_REQUIRED`). Cycles in `blocks`
are rejected (`INVALID: dependency cycle`).

**The fourth one cannot be filled in blind, and that is the point.** `noDependency` is free text,
so an agent can satisfy it without ever looking; `reviewedSimilar` is a list of issue ids only the
server produces. Retasc scans your title AND body for issue mentions and for rare shared words,
and refuses the create while any open issue looks like the same or adjacent work —
`DUPLICATE_CHECK_REQUIRED`, which names them. Most creates never see it: with nothing that
looks alike there is nothing to acknowledge. When there is, two ways through, both fine:

- **File, read the refusal, re-call** with the ids it named. One extra round trip.
- **Search first** (`list_issues`) and name them yourself. Passes in ONE call.

Read them before you name them: the gate proves you were shown the candidates, never that you
judged them right. If one of them **is** this work, do not create it — comment on it, or claim
it. If one is adjacent, create and link it with `add_relation` after. A `blockedBy` on the create
also counts as having read that issue — but declare one only where the dependency is real: an
invented `blocks` edge clears the gate and then leaves your new issue undispatchable until the
other one closes. The
retry recomputes rather than trusting your list, so a look-alike another agent filed in between
is surfaced again. Valid on a create only; on an update it is rejected.

When a human describes what they are working on, file it as issues: one `work:true` issue per
unit an agent can finish, a `work:false` container only when several belong together, `blocks`
edges where order matters, priorities 1..4, and a body with context and acceptance criteria.
Read the plan back to them before filing.
