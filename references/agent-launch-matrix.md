# Agent selection and launch matrix

Two separate decisions: **which agent** should do the work, and **how** to launch it in Herdr.

## Choosing the agent

### Mandatory priority (user-configured, applies when the user does not name an agent)

| Task type | Order of preference |
|---|---|
| 前端开发 (frontend: pages, components, styles, interactions, front-end build/dev server, HTML/CSS/JS/TS/React/Vue/Electron…) | **1. AGY** (required) |
| 非前端开发 (everything else) | **1. OpenCode → 2. CodeBuddy → 3. AGY** (fallback) |

Fallback order is used only for *capability/availability* failures, with evidence recorded. Free-quota
exhaustion (`Free usage exceeded`), `Model is unavailable`, and repeated startup hangs count as
"agent unavailable" — record the error text, then move to the next agent instead of cycling models
inside the same CLI. See `SKILL.md` → "子智能体自动分配规则" for the full execution principles.

### Fit-based defaults (secondary guidance, never overrides the priority above)

Prefer the agent the user names. When the user leaves it open, choose by strength of fit and say
why in one line:

| Work | Good default | Why |
|---|---|---|
| Multi-file refactor inside one repo | Codex | strong at long-horizon code edits and test loops |
| Careful surgical patch with review-like reasoning | Claude Code | good at reading context and explaining trade-offs |
| Quick autonomous exploration / scaffolding | AGY | fast, autonomous by default |
| Repository-wide Q&A, docs, prompt drafting | Claude Code or Codex | thorough reading, good writing |
| Bulk mechanical edits, lint/test grind | Codex | reliable loop execution |
| Second opinion / adversarial review | a *different* agent from the one that wrote the code | independence catches different bugs |
| Work already in progress in an existing pane | that agent (reuse) | keeps context, avoids duplicate efforts |

Rules: one owner per file area when running in parallel; never let two agents edit the same files
concurrently; give the reviewer read-only scope when its job is review.

## Launch mapping

"Kind" is the value for `herdr agent start --kind`. Flags after `--` are passed to the native CLI.

| Agent | Herdr kind | Launch | Autonomous flag status |
|---|---|---|---|
| AGY | `agy` | `herdr agent start <n> --kind agy --pane <p> -- --dangerously-skip-permissions` | supported (`agy` 1.4.7) |
| Claude Code | `claude` | `herdr agent start <n> --kind claude --pane <p> -- --dangerously-skip-permissions` | supported (Claude Code 2.1.310) |
| Codex | `codex` | `herdr agent start <n> --kind codex --pane <p> -- --yolo` | accepted but **hidden** (not in `--help`) in codex-cli 0.164.0; documented alternative `--dangerously-bypass-approvals-and-sandbox`, which is not equivalent in intent — prefer `--yolo` since it is accepted, and say so if a future version rejects it |
| CodeBuddy | none | `herdr pane run <p> "codebuddy --dangerously-skip-permissions"` | supported (2.168.2); note it still asks for HIGH/CRITICAL |
| OpenCode | `opencode` | `herdr agent start <n> --kind opencode --pane <p> -- --auto` | `--auto` exists (1.19.3) but only auto-approves what is not explicitly denied; do not describe it as full bypass |
| Gemini CLI | `gemini` | `herdr agent start <n> --kind gemini --pane <p>` | check `gemini --help` for the current approval flag |
| pi | `pi` | `herdr agent start <n> --kind pi --pane <p>` | check `pi --help` |
| Others | see `herdr agent start --help` kind list | `agent start` if a kind exists, else `pane run` | inspect `<cli> --help` for approval/sandbox flags |

Verified kinds in Herdr 0.9.3: `pi claude codex gemini cursor devin agy cline omp mastracode
opencode copilot kimi kiro droid amp grok hermes kilo qodercli qwen letta maki muse`.
`codebuddy` is not among them → `pane run`.

## Two launch paths

**Path A — supported kind (preferred).** Herdr validates that the agent actually started and is
ready for input, and gives you `agent prompt`, `agent read`, `agent wait`, `send-keys`:

```bash
pane=$(herdr pane split --current --direction right --cwd "$PWD" --no-focus | jq -r '.result.pane.pane_id')
herdr agent start reviewer --kind codex --pane "$pane" -- --yolo
herdr agent prompt reviewer "$(cat .agent-orchestrator/prompts/T-001.md)" --wait --timeout 900000
```

**Path B — CLI without a kind.** Run the command in a pane and drive it through the pane surface:

```bash
pane=$(herdr pane split --current --direction right --cwd "$PWD" --no-focus | jq -r '.result.pane.pane_id')
herdr pane run "$pane" "codebuddy --dangerously-skip-permissions"
herdr pane wait-output "$pane" --match "<startup marker>" --timeout 60000
herdr pane read "$pane" --source recent-unwrapped --lines 80
```

Path B has no `agent prompt`/`agent wait`: send the prompt with `pane send-text` + `send-keys
Enter` (or `pane run`), and verify delivery by reading the pane. Agent lifecycle states are not
available — rely on output evidence only.

## Pre-launch flag verification

Never launch on a remembered flag. Per agent, once per session:

```bash
command -v <cli> && <cli> --version
<cli> --help | grep -iE "dangerous|yolo|auto|permission|sandbox"
```

If the flag is missing: report the exact help output, propose the closest supported alternative
(e.g. CodeBuddy `--permission-mode bypassPermissions`, OpenCode `--auto`), and let the user
decide. Never edit the user's config to force it through.

## Prompt delivery media

1. Short prompt (< ~2k chars): pass inline to `agent prompt`.
2. Long prompt: write it to `.agent-orchestrator/prompts/<task-id>.md`, then prompt with
   `Read and follow the instructions in <abs-path>. Reply DONE when you have read it.`
3. Never split one logical prompt across several messages; the sub-agent must receive one
   self-contained instruction set.
