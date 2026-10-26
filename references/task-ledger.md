# Task ledger and reporting

The ledger is how the orchestrator keeps step 9 ("track delegated tasks") honest. Keep it on disk,
not in your head, and update it at every state change.

## Where

- Project-scoped (default): `<working-dir>/.agent-orchestrator/tasks.md` plus
  `.agent-orchestrator/prompts/<task-id>.md` for long prompts.
- Cross-project / no obvious repo: `~/.agent-orchestrator/tasks.md`.
- Add `.agent-orchestrator/` to `.gitignore` unless the user wants it committed.
- Create it before the first prompt is sent; never backfill silently.

## Index file (`tasks.md`)

```markdown
# Orchestrator task ledger

| ID | Objective | Agent | Kind/surface | Pane | Status | Updated |
|----|-----------|-------|--------------|------|--------|---------|
| T-001 | Fix flaky auth test | codex / api-fix | codex | w6:p9 | running | 2026-10-26 21:40 |
| T-002 | Review auth PR | claude / reviewer | claude | w6:p11 | blocked | 2026-10-26 21:41 |
```

## Per-task block (in the same file or a per-task file)

```markdown
## T-001 — Fix flaky auth test

- Objective: <one line>
- Agent / launch: codex — `herdr agent start api-fix --kind codex --pane w6:p9 -- --yolo`
- Working directory: /abs/path
- Prompt: see prompts/T-001.md (verbatim copy kept)
- Status: running | blocked | done | failed | cancelled
- Acceptance criteria: <copied from the prompt>
- Evidence: `agent get` → working; `agent read` shows plan; `npm test` → 42 passed
- Errors: <verbatim, with the time observed>
- Follow-ups: <message + time + result>
- Next step / decision needed: <what the user must decide>
```

## Status vocabulary

- `planned` — approved, not launched.
- `launching` — pane created / `agent start` in flight, not yet verified.
- `running` — verified `working` or output shows active work.
- `blocked` — waiting on an approval/question dialog; user input required.
- `done` — result read and checked against acceptance criteria.
- `failed` — agent errored, exited, or produced a unusable result.
- `cancelled` — user or orchestrator stopped it.

Only `done` may be reported to the user as complete, and only with the evidence recorded.

## Report format to the user

```markdown
## Status
- T-001 (api-fix, codex) — done. `npm test` 42 passed; files: src/auth/session.ts, tests/auth.test.ts
- T-002 (reviewer, claude) — blocked: asks whether to flag the public API rename as breaking.
  Need your call to continue.

## Verified
<what was actually read/checked, with command → result>

## Not verified / risks
<claims not yet checked, known gaps>

## Proposed next step
<one recommendation, or the decision you need from me>
```

Keep it scannable: one line per task, evidence next to the claim, decisions called out
separately. Never report a task as done because the agent said so — the evidence line must exist.
