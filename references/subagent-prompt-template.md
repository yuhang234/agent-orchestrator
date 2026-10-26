# Sub-agent prompt template and quality bar

A sub-agent starts with zero context. Every prompt must be self-contained, unambiguous, and
checkable. If a stranger cannot execute it and prove the result, it is not ready.

## Template

```markdown
# Task <ID>: <one-line objective>

## Role
You are <role> working in <repo/product>. <One sentence on why this task matters.>

## Context
- Working directory: <abs path>
- Stack: <language/framework/version>, package manager, test runner
- Key paths: <entrypoints, modules that matter>
- Current state: <what exists today, what changed recently>
- Prior attempts / known pitfalls: <verbatim errors, failed approaches>

## Objective
<What must be true when this is done. Outcome, not activity.>

## Scope
In scope:
- <file/area/behaviour>

Out of scope (do not touch):
- <file/area/behaviour>

## Constraints
- <API/back-compat, performance budget, style rules, dependencies not to add>
- Do not modify <X>. Do not commit, push, or change credentials.
- Ask before any destructive or irreversible action.

## Expected outputs
- <code changes / new files>
- <report: what changed, why, and how it was verified>
- <commands run with their key output lines>

## Acceptance criteria
1. <objective, checkable criterion — e.g. `npm test` passes, exit 0>
2. <behaviour criterion — e.g. endpoint returns 400 with body {...} on empty input>
3. <regression criterion — e.g. existing tests untouched and green>

## Verification you must run
```bash
<exact commands>
```

## Reporting format
Reply with: STATUS (done | blocked | partial), WHAT CHANGED (file list + one line each),
VERIFICATION (command → result), OPEN QUESTIONS, RECOMMENDED NEXT STEP.
Keep it under ~30 lines. If blocked, state exactly what you need from me and stop.
```

## Quality bar (checklist before sending)

- [ ] Objective is an outcome with a test, not "look into X".
- [ ] Context contains every fact the agent needs without asking (paths, commands, errors verbatim).
- [ ] Scope has an explicit *out of scope* list.
- [ ] Constraints name what must not change and what needs approval.
- [ ] Acceptance criteria are independently checkable by a third party (you).
- [ ] Verification commands are copy-pasteable and already known to work.
- [ ] Reporting format limits the reply so results are cheap to collect.
- [ ] No secrets, tokens, `.env` contents, or private keys anywhere in the prompt.
- [ ] The prompt names the working directory absolutely.
- [ ] If it involves multiple repos/services, each one is listed with its path.

## Review / analysis prompt variant

For read-only work (review, investigation, second opinion), replace "Expected outputs" with:

```markdown
## Expected outputs
- A findings list: severity, file:line, problem, why it matters, suggested fix
- Explicitly separate "definite bug" from "possible issue" from "style nit"
- Do not modify files. Do not run commands that change repository state.
```

## Delegation anti-patterns

- Bundling unrelated goals into one prompt ("also fix the docs and update CI").
- Vague verbs: improve, clean up, modernise, optimise — always quantify or bound.
- Asking for "a plan" when you need a result, or a result when a plan is what you must approve.
- Reusing a stale prompt after the user changed the requirement; re-draft and re-approve.
- Prompts that assume context from this conversation; the sub-agent has none.
- Telling the agent to "figure out the tests" instead of naming the command.
