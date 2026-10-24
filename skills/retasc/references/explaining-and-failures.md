# Explaining Retasc, and known failure modes

Part of the Retasc skill. [SKILL.md](../SKILL.md) is the overview; section numbers (§N) are shared across these files.

## 16. Explaining Retasc to a confused human

- **"Where do my issues live?"** In a project inside your org, on the server. Not in the repo.
  The folder your agent runs in is *wired to* one project; that is the only link.
- **"Folder vs project."** A project is where work *lives*. A folder is where work is *done*.
  Filling a project (import, filing issues) and connecting a folder to it (`bind`) are two
  separate acts, on purpose.
- **"What is a key, what is an agent?"** Each folder gets one workspace key, which *is* the
  agent member for that folder, acting on your behalf. Each conversation gets a child session
  key so parallel sessions are told apart. The Dash shows agents, not keys.
- **"Assigned vs claimed."** Assigned = which human is responsible (a routing lane). Claimed =
  which agent session is executing it right now, under a lease (30 minutes behind the watchdog,
  2 hours without one). Unassigned work is the shared pool; assigned work is pulled only by
  that human's agents unless you say otherwise.
- **"Why is it `todo` again? I was working on it."** The lease lapsed (nothing renewed it) and
  the reclaimer freed it. Nothing is lost: the checkpoint and comments are on the issue and the
  next claim resumes from them. Behind the stock proxy this only happens on process death,
  sleep, or key rotation.
- **"Blocked vs blocked-by."** The `blocked` *status* is a manual flag for waiting on something
  outside. Dependency blocking is the `blocks` edges, computed live, never shown in the status.
  `queue_status` lists what is dependency-blocked and by what.
- **"Review vs done."** `review` is "finished, needs a second person's acceptance"; it still
  blocks dependents and needs a different human named as reviewer. `done` is terminal and
  unblocks. A solo org skips review.
- **"Why did my agent say the queue is empty when I can see issues?"** They are in another
  person's lane (`otherLanes`), dependency-blocked, held by another session, containers
  (`work:false`), or external intake waiting for a signature (`awaitingTriage`). The pull
  response says which.
- **"Why restart?"** MCP clients read their server list at launch. Until you restart, the tools
  do not exist in the session. Resume is not a restart.
- **"Why can't my agent approve the GitHub-filed issue?"** Its text was written by a stranger,
  and the agent would be the target of anything hostile in it. A person signs it once, in the
  Dash.
- **"What does it cost?"** Per action, fractions of a cent; the meter is `usage_summary`, the
  bill is `billing_summary` or the Dash. Members are free. $10 is seeded at signup.
- **"Can I get my data out?"** Full org export (JSON / CSV) from the Dash, any time. Imports are
  one-way in; GitHub / GitLab is the only place Retasc writes back (closing the source on done).

## 17. Known failure modes, so no human debugs

| Symptom | Cause | Fix |
|---|---|---|
| Only `setup_status` is listed | No credential in this folder | §8 step 1 |
| Tools listed, every call `UNAUTHORIZED` | Credential refused | §8 step 2 |
| Calls worked, then started returning `UNAUTHORIZED` | Key revoked or rotated, or the agent retired, mid-session. (A session key the server retired for going unused or silent a week, `session key expired`, is replaced by the proxy and the call retried since RTSC-1194, so the agent rarely sees that one. Every other refusal is never routed around; a session whose key was retired before RTSC-1194 shipped needs one restart.) | §8 step 2 — relay the two roads the error carries; never invent a cause |
| "Signed in, but the server is unreachable" | Folder not bound; sign-in is global, the binding is per folder | `retasc bind` here, then restart |
| Ran `bind`, tools still missing | Not restarted, or resumed instead of restarted | Cold restart the client |
| `done` fails `CLAIM_MISMATCH` after a resume | New session | `claim_issue` with the old token |
| Issue reclaimed mid-work | Nothing renewing (no proxy AND no session id echoed, or the claim predates this proxy) | `check_claim` twice; `checkpoint` yourself; or re-claim with the token so the proxy adopts it |
| Two agents' edits on one branch | Same checkout | One worktree per claim, before the first edit |
| Agent reports an empty queue, the human sees work | Lanes, blockers, or quarantine | Read the pull's counts and `tellHuman` |
| Agent tries to read a key from disk to upload a file | Followed the wrong tool | `save_attachment_file`; never hunt for credentials |
| Import mapped a column wrong; work vanished from dispatch | The mapping was inferred | Re-import with the human's column-by-column answers |
| A second org appeared on billing | `bind` was run instead of `join` | No CLI delete; the Dash or support |
| Codex / Cursor / Gemini has no Retasc tools | `setup` never ran after that harness was installed | `retasc setup`, restart |
| A wall of bogus type errors in a fresh worktree | Untracked dependencies | Run the repo's install step |
| No Retasc tools at all inside a git worktree | CLI older than 1.45.0: the binding lookup stopped at the worktree's `.git` file | `npm i -g @retasc/cli@latest`, restart. Do NOT re-bind the worktree, that mints a second agent |
| `bind` seems to hang with no output | It is waiting on the browser click | Post the approve URL; it prints before the wait |
| `retasc (ENOENT)` at session start, or `retasc: command not found` | A marker written before 1.49.0 names a bare `retasc` this machine never installed | Re-run `bind` in that folder — it writes the launcher that machine can actually start. Do NOT hand-write `@latest`; see the row below. §9 |
| Tools work but a feature shipped releases ago is missing, with no error | The session started before the update landed (it keeps the build it started with), auto-update is off, or the marker is pinned to an old version such as `@retasc/cli@1.56.0` | Restart the session first. Still old: `retasc doctor` says whether auto-update is on and what it last did; `npx -y @retasc/cli@latest update` moves pinned markers to `@latest`, then restart. `npm i -g` alone does NOT fix a pinned npx marker: the global binary and the npx cache are different copies |
| MCP loads but every call is unauthenticated, in a container | The marker started, but this machine has no keystore and no `RETASC_MCP_KEY` | `npx -y @retasc/cli@latest bind --json`, relay the code to your human, run it again after they approve. §9, shape A1 |
| Pi is running and has no Retasc tools | No MCP extension installed in Pi, or Pi was not restarted after installing one | Pi ships no MCP client; install one (`pi install npm:pi-mcp-adapter`, or whichever they prefer) and restart Pi. §0 |
| **You ARE Pi** and have no Retasc tools | Same cause, and you can fix it: you have a bash tool | Install an extension yourself, then ask your human to restart you — a running Pi cannot load one mid-session. Never replace an extension they already chose. |
| A Pi agent lost its claim after sitting idle for ~10 minutes | `pi-mcp-adapter`'s default lifecycle closes an idle server, killing the watchdog | `retasc setup` (or any Pi session on a current CLI) writes `"lifecycle": "lazy-keep-alive"` into `~/.pi/agent/mcp.json`. §9 |
| Codex or Grok has the remote key in its config and every call is `UNAUTHORIZED` | The other tool's header key: Codex reads `http_headers`, Grok reads `headers`, and each ignores the other's without a word | Use the block `retasc key mint` prints for THAT tool |
