# Herdr CLI reference (verified against 0.9.3, 2026-10)

This file records the surface that the orchestrator actually uses. The installed binary remains
the authority: when something looks different, run `herdr --help` and the relevant group
(`herdr agent`, `herdr pane`, `herdr tab`, `herdr workspace`) and update this file.

Installed binary in the authoring environment:
`~/.herdr/packages/standalone/releases/0.9.3-x86_64-pc-windows-msvc/herdr` (on `PATH`).

## Preconditions

```bash
test "${HERDR_ENV:-}" = 1   # otherwise: not inside Herdr, stop
herdr status                # client/server version + protocol compatibility
```

`herdr status` output on the authoring machine: client 0.9.3, server running, protocol 22,
`endpoint_compatible: yes`.

Do not run bare `herdr` for discovery — it attaches the TUI. Do not probe a mutating nested
command by omitting its arguments (`herdr workspace create` with defaults really creates one).

## Layout and discovery

```bash
herdr workspace list
herdr tab list --workspace "$HERDR_WORKSPACE_ID"
herdr pane current --current
herdr pane list --workspace "$HERDR_WORKSPACE_ID"
herdr pane layout --pane "$HERDR_PANE_ID"     # decide split direction
herdr agent list
```

Public ID shapes: workspace `w1`, tab `w1:t1`, pane `w1:p1`. Parse IDs from JSON responses; they
are never derivable from sidebar order.

Creation responses:

| Command | Path to the new object |
|---|---|
| `workspace create` | `.result.workspace`, `.result.tab`, `.result.root_pane` |
| `tab create` | `.result.tab`, `.result.root_pane` |
| `pane split` | `.result.pane` (`.result.pane.pane_id`) |
| `pane move` | `.result.move_result.pane.pane_id` |

Extraction with `jq` (available on the authoring machine):

```bash
pane=$(herdr pane split --current --direction right --cwd "$PWD" --no-focus | jq -r '.result.pane.pane_id')
```

### Layout tree structure & hierarchy (verified in 0.9.3)

`herdr pane layout --pane "$HERDR_PANE_ID"` returns the exact binary layout tree:
- Hierarchy: `Workspace` (e.g. `w6`) contains `Tabs` (e.g. `w6:t9`); each tab contains a binary `Splits Tree` of `Panes` (e.g. `w6:p9`, `w6:pR`, `w6:pS`).
- `.result.layout.area`: `{ "height": H, "width": W, "x": 0, "y": 0 }` — total tab dimensions.
- `.result.layout.panes[]`: array of panes, each having `pane_id`, `rect: {x, y, width, height}`, and `focused: bool`.
- `.result.layout.splits[]`: array of binary split nodes, each having `id` (e.g. `split_0_root`), `direction` (`right`|`down`), `ratio` (e.g. `0.5`), and bounded `rect`.
- `.result.layout.zoomed`: `true|false` indicating whether any pane is temporarily maximized.
- **Bounding box isolation rule**: A split partitions *only* the bounding box of the target pane. Splitting an existing child pane leaves all sibling branches (specifically the main agent's rect) completely unchanged.

## Pane commands

```
herdr pane  list current get layout process-info neighbor edges focus resize zoom read
            rename input split swap move close send-text send-keys wait-output run
            report-agent report-agent-session release-agent report-metadata
```

Split:

```bash
# 1. First sub-agent: split the caller's main pane (wide screen: right, tall: down)
herdr pane split --current --direction right --cwd "$PWD" --no-focus
# Returns .result.pane.pane_id (e.g. w6:pR)

# 2. Second sub-agent: split the EXISTING child pane (orthogonal direction, e.g. down)
herdr pane split --pane <CHILD_PANE_ID> --direction down --cwd "$PWD" --no-focus
# Crucial: splitting the child pane bisects ONLY the child's bounding box,
# leaving the main caller pane's area (ratio 0.5, left half) completely intact!
```

Options: `--pane <ID>`, `--current`, `--direction right|down`, `--ratio <FLOAT>`, `--cwd <PATH>`,
`--env KEY=VALUE`, `--focus`, `--no-focus`, `--right-click herdr|pane`.

Zoom (temporary overlay inspection without changing layout tree):

```bash
# Toggle zoom on a pane (maximize to full canvas without altering the layout tree)
herdr pane zoom --pane <pane-id> --toggle
# Or explicitly turn on/off:
herdr pane zoom --pane <pane-id> --on
herdr pane zoom --pane <pane-id> --off
```

Options: `--pane <ID>`, `--current`, `--toggle`, `--on`, `--off`.
*Note: Zoom is a pure display overlay. Exiting zoom with `--off` or `--toggle` restores the exact prior multi-pane rects.*

Run a command and read output:

```bash
herdr pane run <pane-id> "just test"
herdr pane wait-output <pane-id> --match "test result" --timeout 120000
herdr pane read <pane-id> --source recent-unwrapped --lines 120
```

## Agent commands

```
herdr agent  list get read send-keys prompt rename focus wait attach start explain
```

Supported kinds for `agent start` (0.9.3):

```
pi claude codex gemini cursor devin agy cline omp mastracode opencode copilot kimi kiro
droid amp grok hermes kilo qodercli qwen letta maki muse
```

`codebuddy` is **not** a kind — launch it through `pane run`.

Start:

```bash
herdr agent start <NAME> --kind <KIND> --pane <PANE_ID> [--timeout <MS>] [-- <agent-args>...]
```

- The pane must be an available interactive shell at its prompt.
- Returns success only after the expected agent is detected in the same pane and is ready for
  input; default readiness timeout 30s (max 300s).
- If the agent is blocked during startup it returns `agent_not_ready` and keeps the name usable
  for `agent read` / `agent send-keys`. Wait for idle before prompting.

Prompt:

```bash
herdr agent prompt <TARGET> <TEXT> --wait --timeout 900000
herdr agent prompt <TARGET> <TEXT> --wait --until blocked --timeout 120000
```

- `<TARGET>` is a unique live agent name or the pane ID hosting it (not a terminal ID, not a
  bare kind label).
- Rejected with `agent_blocked` when the agent is already at an approval/question dialog, before
  any input is written.
- With `--wait`, submission from a non-working state must show `working`/`blocked` within 5s, else
  `agent_prompt_stalled`; caller timeout expiry returns `timeout`.
- Successful submission means text + Enter were written; it does **not** prove the turn started.
  Confirm with `agent get` / `agent read`.
- `idle` and `done` both mean ready for input.

Wait / inspect:

```bash
herdr agent wait <TARGET> --until blocked --timeout 120000
herdr agent get <TARGET>
herdr agent read <TARGET> --source recent-unwrapped --lines 120 [--format ansi]
```

Read sources: `visible` (viewport), `recent` (soft wraps kept), `recent-unwrapped` (preferred for
logs/transcripts), `detection` (plain bottom snapshot used for agent detection).

Keys:

```bash
herdr agent send-keys <TARGET> esc
herdr agent send-keys <TARGET> ctrl+c
```

Keys are validated before any byte is written. Use `pane send-text` / `pane send-keys` only when
raw terminal control is intentional.

Lifecycle states: `idle`, `working`, `blocked`, `done`, `unknown`. `unknown` means an agent is
present but unclassified — it never proves completion.

## Error codes to expect

| Code | Meaning | Correct response |
|---|---|---|
| `agent_not_ready` | agent blocked/slow during startup | inspect with `agent read`, wait for idle, then prompt |
| `agent_blocked` | approval/question dialog open | read the UI, ask the user, then answer or interrupt |
| `agent_prompt_stalled` | prompt written, no activity observed in 5s | read the pane before resending |
| `timeout` | caller timeout expired | read state; do not assume non-delivery |
| `agent_not_found` / ambiguity | bad or reused name | re-resolve from `agent list` |
| `workspace_group_close_required` | closing a grouped workspace | ask the user; do not bypass with `--group` |

Exit status: JSON error on stderr with status 1 (server error), status 2 (CLI syntax error).

## Recovery tactics

- Prompt too long to paste safely: write it to a file, then prompt with
  `Read and follow the instructions in <path>`.
- Response not recoverable from `agent read` (alternate-screen UIs): ask the agent to write a
  Markdown summary to a temp path and reply with the path, then read that file on the same
  machine. Use only as a fallback, never as the default instruction.
- Worktree isolation: `herdr worktree` helpers exist; use them only when the user asks for
  isolated worktrees. `--trust-repository` only after the user verified the repository.

## Never

- `herdr server stop`, killing the Herdr process, or closing other clients' workspaces.
- Closing tabs/panes/workspaces this session did not create.
- `--machine` combined with `--session`/`--remote`; forwarded IDs and `--current` do not apply on
  a remote machine — rediscover IDs there with the same `--machine` prefix.
