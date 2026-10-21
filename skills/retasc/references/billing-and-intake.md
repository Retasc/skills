# Billing, intake, imports, ghosts and connectors

Part of the Retasc skill. [SKILL.md](../SKILL.md) is the overview; section numbers (§N) are shared across these files.

## 10. Billing (what a call costs, and what refuses)

Per action, in micro-USDC (1,000,000 = $1). Signup seeds **$10 once per user** (their first
org only). Credit is bought in the Dash.

| Tier | Tools | Price |
|---|---|---|
| Dispatch | `next_issue`, `next_batch` (claiming), `claim_issue`, `add_relation`, `remove_relation` | $0.007 |
| Work | `save_issue` create or body edit, `checkpoint`, `save_comment` | $0.005 |
| Bookkeeping | `save_issue` status / priority / title / other edits, `save_label`, `save_attachment`, `obsolete_attachment`, `retract_comment` | $0.003 |
| Plumbing | `mint_session_key`, `record_session`, `name_workspace`, `release_issue`, `revoke_connector`, every read (a `next_batch` peek included) | $0.001 |
| Heartbeat | `heartbeat`, and a re-claim of an issue this session already holds (a `claim_issue` that answers `renewedOnly: true`) | $0.0001 |
| Files | upload $0.003 + $0.001/MB; download $0.0005/MB per read | size-priced |
| Free | `invite_member`, `list_invites`, `revoke_invite`, `accept_invite`, `suspend_member`, `reactivate_member`, `retire_member`, `usage_summary`, `billing_summary`, `report_worktrees`, `mint_harness_key` | $0 |

**Gate**: an org that is `needs_reauth`, `canceled`, or `lapsed`, or over its monthly cap,
refuses value-bearing writes with `BILLING_INACTIVE` / `MONTHLY_CAP_REACHED` and a
`tellHuman`. Reads are never gated. **Exempt**, so work can wind down: `heartbeat`,
`checkpoint`, `release_issue`, `revoke_connector`, `record_session`, `name_workspace`,
`report_worktrees`, `mint_harness_key`, and every membership tool. You cannot fix a billing refusal; only an owner can. Do not keep
working read-only as if the session were healthy.

## 11. Intake, quarantine, imports, ghosts, connectors

- **Connectors** (GitHub Issues, GitLab) sync external issues and PRs in, and close them at the
  source on `done` / `canceled`. Connecting one is Dash-only (it takes a raw token). Over MCP:
  `list_connectors`, `revoke_connector`.
- **Quarantine**: every connector-created issue is `AWAITING_TRIAGE` until a **human** reads it
  and signs it (Dash, or `retasc triage <id>` in a real terminal; there is deliberately no
  `--approve` flag). The signature is bound to a hash of the text, so an upstream edit
  re-quarantines. No agent path approves. Dispatch counts it as `awaitingTriage` and relays a
  `tellHuman`.
- **Imports** (Linear, Jira, Asana, ClickUp, Shortcut): one-way in, one at a time per org,
  cannot be undone. Over MCP, in this order: `list_import_sources` (ask for each `authFields`
  by its label; secrets pass through the conversation, say so, never echo them) →
  `list_import_targets` → `list_import_statuses` (**ask the status mapping column by column;
  never infer it from names**) → `list_review_candidates` for any column mapped to `review` →
  `import_history` (warn on a re-import: it overwrites Retasc-side edits) → `run_import` with
  `destination` = the human's answer (`this_project`, or `new_project`, which is invisible from
  this folder; say so) → poll `latest_import`. Confirm in plain words before `run_import`.
  From a terminal: `retasc import`.
- **Ghosts**: an import mints login-less placeholder members for the source's authors. A human
  claims theirs once (`list_claimable_ghosts` → `claim_ghost` with the id **they** chose;
  `retasc identity`; or the Dash). A wrong claim re-attributes someone else's work permanently.
