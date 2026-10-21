# Every MCP tool

Part of the Retasc skill. [SKILL.md](../SKILL.md) is the overview; section numbers (§N) are shared across these files.

## 12. Every MCP tool, by purpose (58)

Descriptions are self-describing at runtime; this is the map.

**Identity and setup**: `whoami` · `setup_status` · `mint_session_key` (proxy) ·
`mint_harness_key` (proxy) · `name_workspace` (CLI) · `record_session` (proxy)

**Dispatch and lease**: `next_issue(allLanes?)` · `next_batch(n?, claim?, allLanes?)` ·
`claim_issue(identifier, claimToken?)` · `release_issue(identifier, claimToken?, note?)` ·
`heartbeat(identifier, claimToken?)` · `checkpoint(identifier, note, claimToken?)` ·
`check_claim(identifier)` · `queue_status()` · `report_worktrees(worktrees)` (your PROXY
calls this, not you — see §15)

**Issues**: `save_issue(...)` (create or update; §3, §6) · `get_issue(identifier)` (body,
labels, relations, `blockedByOpen`, deadline surface, `claim{heldBy, expiresAt, youHold}`) ·
`list_issues(status?, priority?, label?, author?, assignee?, slaState?, limit?)` (active work
by default; `author` / `assignee` accept `me`; `assignee` accepts `none`; default 50, max 200)

**Graph**: `add_relation(sourceId, targetId, type)` · `remove_relation(sourceId, targetId,
type)`. `blocked_by` is stored as the inverse `blocks`; `related` is canonicalized to one row.

**Memory**: `save_comment(issue, body)` · `list_comments(issue)` · `retract_comment(comment,
note)`

**Attachments**: `save_attachment(issue, url, title?)` (a link) · `save_attachment_file(issue,
filename, contentBase64 | path)` (≤ 256 KiB inline; any size through the proxy's `path` form)
· `list_attachments(issue)` · `get_attachment(attachment)` · `get_attachment_file(attachment)`
(≤ 128 KiB text or ≤ 2 MiB image inline; any size via the proxy) ·
`obsolete_attachment(attachment, reason)` · `prepare_attachment_upload(issue)` (for a human or
script that already holds a raw key; **you do not, so never call it**)

**Labels and projects**: `save_label(name, color?)` · `list_labels()` · `list_projects()` ·
`get_project(prefix?)`

**People**: `list_members(query?)` · `invite_member(...)` (**call it with no arguments first**;
it returns the questions; then again with the answers and `decided:true`; never unprompted) ·
`list_invites()` · `revoke_invite(inviteId)` · `accept_invite(org)` (only after the human says
yes) · `suspend_member(memberId)` (people) · `reactivate_member(memberId)` ·
`retire_member(memberId)` (agents, terminal)

**Imports and ghosts**: `list_import_sources` · `list_import_targets` · `list_import_statuses`
· `list_review_candidates` · `import_history` · `latest_import` · `run_import` ·
`list_claimable_ghosts` · `claim_ghost(memberId, name)` · `dismiss_ghost_prompt()`

**Connectors**: `list_connectors()` · `revoke_connector(connectorId)`

**Money**: `usage_summary()` (the meter) · `billing_summary()` (owner: the full picture)

**Analytics**: `analytics_overview(days?, from?, to?, tz?, maxBars?, project?, allProjects?,
person?, sections?, cursor?)` (every number on the Dash's Analytics page, as data, with units
and a definition per metric; build the chart yourself) · `analytics_issues(by?, days?, from?,
to?, project?, allProjects?, person?, limit?, cursor?)` (the per-issue timestamps, people,
runtime and cost behind those numbers, paged). Both read THIS key's project unless your
principal is an owner or admin, who may pass another project or the whole org; a member asking
for more is refused, never quietly narrowed. Money is owner/admin, as on the Dash. Past the
inline size cap a page returns a cursor: call again with the same arguments plus it.

Writes that land something a human can be wrong about — claiming an issue, saving one,
commenting, importing, claiming a ghost — carry a footer naming the org and project it
landed in. Bookkeeping writes (`heartbeat`, `checkpoint`, `report_worktrees`) do not:
the footer is there for a human reading the transcript, and nobody reads those.
