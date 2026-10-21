# What the CLI puts on the machine, and containers

Part of the Retasc skill. [SKILL.md](../SKILL.md) is the overview; section numbers (§N) are shared across these files.

## 9. The local architecture (what the CLI puts on the machine)

- **Keystore** `~/.retasc/bindings.json` (mode 0600): workspace id → `{orgId, projectId, key,
  url, prefix, boundPath}`. Keys never live in a project tree.
- **Folder marker**: secret-free. Either `./.mcp.json` carrying `RETASC_WORKSPACE=<ws_id>` in
  the proxy entry's env, or the harness's per-folder "local" scope. There is **no global
  scope** for Retasc; a machine-wide key would leak issues across orgs. `--scope user` is
  refused.
- **`retasc setup`** (once per machine; `bind` runs it the first time): writes ONE `auto` entry
  into every detected harness. The entry names no workspace; the spawned proxy resolves the
  folder it started in against the keystore. One machine-wide wiring, per-folder routing.
  Re-run it after installing a new harness.
- **Pi** (RTSC-1154): `setup` also writes Pi's entry, into Pi's own `~/.pi/agent/mcp.json`
  (or `$PI_CODING_AGENT_DIR/mcp.json`), with `"lifecycle": "lazy-keep-alive"` (started on
  first use, never closed for idleness). Pi ships no MCP
  client, so a human still installs one (`pi install npm:pi-mcp-adapter`); without it the
  entry is inert. The keep-alive is the point: the adapter's default closes an idle server
  after 10 minutes, which killed the watchdog and let your leases lapse while you worked.
  The adapter merges servers field by field, so a project `./.mcp.json` that also names
  `retasc` keeps its workspace and still inherits the lifecycle. A proxy Pi launches adds
  the entry itself when it is missing, so machines set up before this heal on their own.
  Still `checkpoint` at long-silence boundaries and use `check_claim` if in doubt (§5).
- **`retasc mcp-proxy`** (spawned by the harness, stdio): forwards JSON-RPC to the remote MCP;
  mints a **session key** at startup; records the transcript id and model (`record_session`)
  and the folder name (`name_workspace`); **heartbeats** every 10 minutes for claims it saw;
  reaps the worktree and branch when this session closes a merged issue; overrides
  `save_attachment_file` / `get_attachment_file` with `path` forms so files move to and from
  disk at any size. If it dies, heartbeats stop and the server reclaims: fail-safe. In an
  unbound folder it answers "this folder is not bound, run `retasc bind`" instead of a 401.
- **Claude Code hooks** (wired by `setup`): `SessionStart` and `PostModelSwitch` run
  `retasc hook …` so the proxy can label the session. Other harnesses have no hook; their
  sessions carry less. Normal.
- **Hosted agents and CI**: no CLI. A **human** mints a key in their own terminal
  (`retasc key mint --hosted`) or in the Dash and puts it in the runner's secret store; the
  `--hosted` is not optional: it is what tells the server this key will never have a local
  watchdog, so it is asked to renew its own leases instead of being told to run `bind`
  (RTSC-859). The runner sends
  `Authorization: Bearer <key>` to `https://mcp.retasc.com/mcp`. Never run `key mint`
  yourself: its output is a raw, long-lived org credential and would land in your transcript.
  No proxy ⇒ renewal rides on your own calls if you echo the session id, and is entirely
  yours if you don't (§5).
- **Platform**: macOS is the tested platform; `doctor` prints a note elsewhere.

### In a container or cloud session

Everything above assumes a laptop: a `retasc` binary on `PATH` and a home keystore, both
per-machine state a fresh clone does not carry. A container has neither, so the committed
`./.mcp.json` marker fails twice over, first on the binary (`retasc (ENOENT)`) and then,
behind it, on the credential. There are two container shapes, and which one you are in has
nothing to do with which harness you run.

**Shape A, the container can run a process** (Claude Code cloud, CI runners, most agent
sandboxes). The marker starts on its own — since 1.49.0 a committed marker names
`npx -y @retasc/cli@<version> mcp-proxy`, which needs no global install. What is left is
the credential, and there are two ways to get one.

**A1 — sign in from inside the container (RTSC-879, since 1.49.0). No secret anywhere.**
This is the path when a human is in the chat with you, which in a cloud session they are.

1. `npx -y @retasc/cli@latest bind --json`. It starts a device grant, prints an approve
   URL and an eight-character code, and **exits** — it does not wait.
2. Show your human BOTH lines. They open the URL on **any** device, a phone included: the
   approval happens at the provider, so it does not have to be this machine, and nothing
   redirects back here.
3. Run the **same command again** once they say they have approved. It resumes the grant
   it already started, signs in, and finishes binding. Repeat if they were slow; each run
   is bounded and picks up where it left off.

Do NOT start over between attempts. A second `bind` while one is pending issues a fresh
code and invalidates the one they are looking at — the outcome state is `SIGN_IN_PENDING`
precisely so you can tell "waiting on a click" from "nothing has started".

Two doors stay shut, on purpose: **CI**, where nobody is there to approve anything, and
**`RETASC_NO_BROWSER`**, which is a person declining browser auth. Both refuse
immediately with a terminal outcome rather than emitting a code nobody will read.

**A2 — a key from the platform's secret store.** Right for CI and for anything unattended.
A human mints it on their own machine (`retasc key mint --hosted`, or the Dash) and sets
`RETASC_MCP_KEY`, plus `RETASC_MCP_URL` only if the deployment is self-hosted. The proxy
reads that variable *before* the keystore. Ask for the secret; never mint one yourself.

Note the platform may not let you set it. A Claude Code cloud session started from the
desktop button has no environment-variable door the session itself can reach, which is
why A1 exists.

**The key comes from the environment, never from a file.** `.githooks/pre-commit` refuses a
staged `.mcp.json` containing `RETASC_MCP_KEY` or an `authorization` key, because a
credential committed once stays readable in git history long after it is deleted.

Two things that bite in this shape, both worth knowing before you debug them. `setup`
makes the binary real by installing the CLI, so the committed marker's `command: "retasc"`
resolves afterwards; if that install cannot happen (no write access to the npm prefix) the
marker still names a command that does not exist, and in Claude Code a project-scope
`.mcp.json` outranks the user-scope entry `setup` just wrote, so it keeps winning and
keeps failing. Fix it by re-running `bind` in that folder — it rewrites the marker with
the launcher that is right for THAT machine. A hand-written marker should say
`@retasc/cli@latest`, which is what `bind` writes: npx re-checks a tag on every start, so
it stays current, where a pinned version stays on that version (RTSC-1153). And with no key in the environment the proxy still
starts and `setup_status` answers `NOT_CONNECTED` with the instruction, which is a
different failure from ENOENT and means the credential, not the launcher.

**Shape B, an HTTP-only host** (claude.ai connectors, Codex Cloud, a hosted OpenCode). No
process, no environment you control, so no proxy is possible in principle. The entry is the
direct form: the remote URL plus `Authorization: Bearer <key>`. `retasc key mint` prints
all three spellings, each labelled with the harnesses that read it, because they are not
interchangeable: Codex reads an `[mcp_servers.retasc.http_headers]` table and Grok reads
`[mcp_servers.retasc.headers]`, and **each silently ignores the other's**, loading happily
into a server that is enabled, configured and unauthenticated. The only symptom is
`UNAUTHORIZED` at the first tool call. Put the block in the harness's own user-scope config
or the platform's secret store, never in a tracked file. No proxy also means nothing
renews your leases: `heartbeat` or `checkpoint` yourself (§5).
