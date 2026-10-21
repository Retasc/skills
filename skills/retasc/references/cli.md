# Every CLI command

Part of the Retasc skill. [SKILL.md](../SKILL.md) is the overview; section numbers (§N) are shared across these files.

## 13. Every CLI command, by purpose

Install: `npm i -g @retasc/cli`, or run any command through `npx @retasc/cli@latest …`.
`retasc --help` and `retasc <command> --help` are complete and current.

| Command | Purpose |
|---|---|
| `login [--github \| --google]` · `logout` | Human sign-in (device flow). Management commands need it; MCP work does not. |
| `whoami [--json]` | This folder's org / project / agent / session, then the signed-in user and their orgs. |
| `bind [--org-id \| --org-name] [--project-id \| --project --prefix] [--runtime] [-y] [--no-install] [--setup <token>] [--json]` | **The** way to bind a folder to one org + project: mint a workspace key, write keystore + marker, run `setup`. Loud on re-bind. `--json` when an agent drives it. |
| `join <link \| code> [--no-bind] [--project-id] [-y] [--no-install]` | Invited teammate: sign in, redeem, claim identity, bind this folder, wire MCP. One command. |
| `unbind [-y]` | Remove the keystore entry and MCP entry, revoke the key. |
| `doctor` | Is this folder correctly and safely bound; can the launcher start; any illegal global server. |
| `setup [--no-install]` | Wire the `auto` proxy entry into every harness on the machine, once. |
| `update` | Bring this machine up to date now: the global install if there is one, the harness entries, and any folder marker still pinned to an old version (moved to `@latest`). No sign-in, no prompt, any folder. A build that is behind hands over to the newest. Rarely needed: the CLI updates itself (below). |
| `init --project --prefix [--org \| --org-id] …` | Create org + project and bind, in one shot. `bind` is preferred for existing orgs. |
| `org create --name` · `project create --org-id --name --prefix` · `project rename-prefix --project-id --prefix` | Owner management. Rename rewrites every identifier. |
| `key mint --org-id --project-id [--runtime] [--name] [--hosted] [--install]` · `key list --org-id` · `key rotate --key-id` · `key revoke --key-id` | Agent keys for hosted agents / CI or manual wiring. Shown once. `--hosted` marks a key that will never have a local watchdog: it is then asked to self-renew rather than told to run `bind`, and the Agents page says "no folder (hosted)" as a fact (RTSC-859). A human runs `mint` and `rotate`; an agent never does, the output is a raw credential. |
| `members invite [--org-id] [--expires-days] [--project-id …]` · `members list --org-id` · `members revoke --invite-id` | Invite humans. Omitted `--project-id` asks; skipped = all projects. |
| `identity [--org-id]` | Claim an imported placeholder as yourself. Never scriptable. |
| `import [--org-id] [--source] [-y]` | Bring a tracker across from the terminal; the mapping is always asked. |
| `triage [issue] [--org-id] [--json]` | List quarantined external work; read and approve one (interactive only). |
| `billing [--org-id] [--json]` | Subscription, what is owed, charge and payment history. |
| `claim [issue] [--id] [--all-lanes] [--base] [--dir] [--no-fetch] [--no-worktree] [--shell] [--print-path] [--json]` · `next` | **Human** convenience: claim over the workspace key and drop into a fresh worktree. Agents claim over MCP. |
| `release [issue] --claim-token [--note]` | Hand a claim back. |
| `checkpoint [issue] [--note] [--claim-token]` | Record a handoff note and renew the lease. |
| `check-claim [issue] [--json]` | Does this session hold it; is anything renewing it. Non-zero exit if not. |
| `done [--id] [--force] [--base] [--dry-run] [-y]` | Mark done (holder only) and tear down the merged worktree + branch. |
| `tidy [--prune] [--force] [--only] [--base] [--json] [-y]` | Reconcile `<prefix>-NN/*` branches against issue status; reap done + merged (dry-run by default). |
| `issue show [issue] [--json]` · `issue list [--status] [--priority] [--label] [--author] [--assignee] [--sla] [--limit] [--json]` | Read the queue from a terminal over the folder's key. |
| `gate install [--prefix] [--no-hook] [--no-action]` | Optional commit-msg hook + GitHub Action requiring `PREFIX-NN` or `[no-issue]`. Opt-in process; Retasc never enforces it. |
| `mcp install --key [--scope local \| project] [--url] [--no-watchdog]` | Manual wiring with a raw key (older form; `setup` + `bind` is the current path). |
| `config` | Resolved paths, endpoints, and whether auto-update is on. |
| `config auto-update on\|off` | Automatic updates, ON by default: once a day the watchdog checks npm in the background and, if a newer CLI is out, installs it; it takes effect in the next session, never mid-session. Off in CI, or with `RETASC_AUTO_UPDATE=0`. The choice is kept until changed. `doctor` shows the state and the last update. |

Spawned by the harness, never typed: `mcp-proxy` and `mcp proxy` (hidden from `--help`),
`hook session-start`, `hook model-switch` and `hook prompt` (Cursor's per-prompt hook,
RTSC-1155), listed under `retasc hook --help`.
