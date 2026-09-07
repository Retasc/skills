# Retasc skills

[Agent Skills](https://agentskills.io) for [Retasc](https://retasc.com), the work queue AI coding
agents pull from. One folder per skill, read unmodified by Claude Code, Codex, Cursor, OpenCode,
Gemini CLI and any harness that follows the standard.

| Skill | What it does |
|---|---|
| [`skills/retasc`](skills/retasc/SKILL.md) | Sets Retasc up from nothing installed (CLI, `bind`, restart, `setup_status`), then works the queue correctly: take work, isolate in a worktree, keep the lease alive, checkpoint, hand off through review, recover a claim after a restart. Carries the model the server enforces, every MCP tool and CLI command by purpose, and the fix for every known failure, so the human never debugs the setup. |

## Install

Pick one:

```bash
# any harness (Claude Code, Codex, Cursor, OpenCode, Gemini CLI, …)
npx skills add Retasc/skills
```

```
# Claude Code, with updates via /plugin update
/plugin marketplace add Retasc/skills
/plugin install retasc@retasc
```

```bash
# by hand
git clone https://github.com/Retasc/skills.git retasc-skills
cp -R retasc-skills/skills/retasc ~/.claude/skills/retasc   # or your harness's skills folder
```

Restart the harness afterwards. Then, in the folder you want connected, tell your agent:

> Read the Retasc skill and set Retasc up in this folder.

Everything after that is the skill's job: it installs the CLI if it is missing, runs `bind` for
you, hands you the approve link, asks you the few questions the server needs, and tells you when
to restart.

## Where this comes from

The skill is written next to the code it describes, in Retasc's monorepo, and a test there fails
whenever a tool or command exists in the code without the skill naming it. This repository is a
mirror; every commit names the source revision. Edits belong upstream: open an issue here and it
will be fixed at the source.

## Licence

MIT. See [LICENSE](LICENSE).
