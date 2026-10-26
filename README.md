# agent-orchestrator

A Claude/agent skill that turns a single coordinator session inside **Herdr** into a multi-agent
dispatcher: plan → draft sub-agent prompts → launch CLIs in new panes → verify delivery →
supervise → report verified results.

The operating prompt is fixed. Everything else in this skill is additive operational detail
(verified Herdr CLI commands, launch matrix, prompt template, task ledger).

## Layout

```
agent-orchestrator/
├── SKILL.md                              # operating prompt (fixed) + operating procedure
└── references/
    ├── herdr-commands.md                 # verified Herdr 0.9.3 CLI surface + error codes
    ├── agent-launch-matrix.md            # which agent for which job, and how to launch it
    ├── subagent-prompt-template.md       # prompt template + quality bar + anti-patterns
    └── task-ledger.md                    # ledger format, status vocabulary, report format
```

## Install

The skill is a directory; install it by linking it into the skills directory of each runtime you
use. Adjust paths for your platform (`~/.claude/skills`, `~/.codex/skills`, `~/.agents/skills`,
`~/.config/opencode/skills`, `~/.pi/agent/skills`, …).

Windows (PowerShell, admin or developer mode for symlinks):

```powershell
New-Item -ItemType SymbolicLink `
  -Path "$env:USERPROFILE\.claude\skills\agent-orchestrator" `
  -Target "D:\AData\dev\study\agent-orchestrator"
```

bash / git-bash:

```bash
ln -s /d/AData/dev/study/agent-orchestrator ~/.claude/skills/agent-orchestrator
ln -s /d/AData/dev/study/agent-orchestrator ~/.pi/agent/skills/agent-orchestrator
```

Copying the directory instead of linking also works; linking keeps edits live.

## Requirements

- Herdr installed and a server running; the orchestrator must itself run inside a Herdr-managed
  pane (`HERDR_ENV=1`).
- At least one supported agent CLI on `PATH` (`agy`, `claude`, `codex`, `codebuddy`, `opencode`, …).
- `jq` recommended for parsing Herdr JSON responses.

## First run

1. Start the skill inside a Herdr pane and describe a task.
2. The skill verifies environment, proposes a split with agents and prompts, and waits for
   approval on substantial work.
3. After approval it creates sibling panes, launches agents with their autonomous flags, submits
   the reviewed prompt, then proves delivery with `agent get` / `agent read`.
4. Progress is tracked in `.agent-orchestrator/tasks.md` and reported with evidence attached to
   every claim.

## Maintenance

Herdr is the authority for its own CLI. When behaviour differs from
`references/herdr-commands.md`, run `herdr --help`, `herdr agent`, `herdr pane` and update that
file — the skill is expected to drift-check rather than assume.
