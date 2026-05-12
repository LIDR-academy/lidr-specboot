---
name: run-parallel-tasks
description: Run N feature tasks in parallel, each in its own worktree, following the full specboot pipeline (enrich → new → ff → apply → verify). Stops after verify — no archive, no commit, no cleanup. Explicit task arguments override `parallel-tasks.md`; file is fallback only.
author: LIDR.co
version: 1.1.0
---

# run-parallel-tasks Skill

Resolves tasks from explicit command arguments first, otherwise reads `parallel-tasks.md` from the references subfolder. Then spins up one isolated agent per task and runs each through the full specboot pipeline without supervision.

**Pipeline per agent:** worktree → enrich-us → opsx:new → opsx:ff → opsx:apply → opsx:verify → stop

## When to invoke

- User says "run parallel-tasks.md"
- User says "run the tasks" (when `parallel-tasks.md` exists in root)
- User says "start the parallel tasks"
- User provides explicit task arguments, for example: "run parallel tasks SCRUM-83 SCRUM-84"

## Instructions

### Step 1 — Resolve task source (arguments first, file fallback)

Parse user input for explicit task arguments after the trigger phrase (for example, in `run parallel tasks SCRUM-83 SCRUM-84`, the arguments are `SCRUM-83` and `SCRUM-84`).

If one or more explicit task arguments are present:
- Do **not** read `parallel-tasks.md`.
- Build tasks directly from arguments.
- For each argument `<TASK_ID>`, create:
  - `name: <TASK_ID>` lowercased to kebab-case
  - `us: <TASK_ID>` (treat as Jira source unless it clearly matches a file path)
- Announce: "Using explicit task arguments (overriding file): name-1, name-2, ..."

If no explicit task arguments are present:
- Read `parallel-tasks.md` from the references subfolder.
- Extract every uncommented task block. A task block starts with `### Task` and contains:
  - `name:` — kebab-case change name (required)
  - `us:` — source type: `inline`, a file path, or a Jira ticket ID (required)
  - `description:` — inline US text (required when `us: inline`)
- Skip any task block wrapped in `<!-- -->` comments.
- Announce: "Found N task(s): name-1, name-2, ..."

### Step 2 — Enrich US for each task

**Always run `enrich-us` for every task**, regardless of source format. Enrichment is mandatory.

First, resolve the raw US text by source:
- **`us: inline`** — use the `description:` field as the raw US input.
- **`us: <file-path>`** — read the file at that path as the raw US input.
- **`us: <JIRA-ID>`** — pass the ticket ID directly to `enrich-us`; it fetches and enriches in one step.

Then, for inline and file-path sources, invoke the `enrich-us` skill passing the raw US text as input.
Capture the full enriched output (with all technical detail, endpoints, files, test cases, NFRs) as the enriched US for this task.

Store each enriched US in memory before proceeding to Step 3.

### Step 3 — Spawn one agent per task in parallel

Launch all agents simultaneously using the Agent tool with `run_in_background: true`.

Each agent must receive a **fully self-contained prompt** — agents start cold with no session memory. Use the template below, substituting:
- `{{CHANGE_NAME}}` — the task `name` field
- `{{ENRICHED_US}}` — the full resolved enriched US text from Step 2
- `{{PROJECT_ROOT}}` — absolute path to the project root
- `{{BASE_BRANCH}}` — current git branch (run `git branch --show-current` to get it)

### Step 4 — Wait and report

When all background agents complete, print a summary table:

```
| Task | Worktree | Tasks | Verify | Blockers |
|------|----------|-------|--------|----------|
| name | .worktrees/name | N/N | PASSED / WARNINGS / CRITICAL | none / description |
```

If any agent reported CRITICAL issues or blockers, flag them clearly for the user.

---

## Agent prompt template

```
You are implementing a feature for a full-stack FastAPI + React project.

Project root: {{PROJECT_ROOT}}
Base branch: {{BASE_BRANCH}}

Work exclusively inside the worktree you will create. Never modify the main checkout.

---

## Feature: {{CHANGE_NAME}}

## Enriched User Story

{{ENRICHED_US}}

---

## Pipeline — execute every step in order, do not skip any

### Step 1 — Create isolated worktree

Use the `using-git-worktrees` skill to create a worktree.
- Branch name: `feature/{{CHANGE_NAME}}-backend`
- Worktree path: `{{PROJECT_ROOT}}/.worktrees/{{CHANGE_NAME}}`
- Base: `{{BASE_BRANCH}}`

All subsequent steps run from inside this worktree.

### Step 2 — Create the OpenSpec change

Run from the worktree:
  openspec new change "{{CHANGE_NAME}}"

This scaffolds the change at `openspec/changes/{{CHANGE_NAME}}/`.

### Step 3 — Generate all artifacts (fast-forward)

Use the `opsx:ff` skill for change `{{CHANGE_NAME}}`.
Provide the Enriched User Story above as the primary source of truth for all artifacts.
Generate: proposal → design → specs → tasks in one pass.

### Step 4 — Implement all tasks

Use the `opsx:apply` skill for change `{{CHANGE_NAME}}`.
Work through every task in `tasks.md`. Do not stop until all tasks are marked `[x]`.

Rules:
- If a task requires a live DB or backend, start the required services first.
- After any CREATE/UPDATE/DELETE curl or Playwright test, restore the data state.
- Mark each task `[x]` immediately after completing it.

### Step 5 — Verify

Use the `opsx:verify` skill for change `{{CHANGE_NAME}}`.
Record all CRITICAL, WARNING, and SUGGESTION findings.

### Step 6 — Stop and report

Do NOT archive, commit, push, or remove the worktree.

Return this exact summary (fill in values):

TASK: {{CHANGE_NAME}}
WORKTREE: {{PROJECT_ROOT}}/.worktrees/{{CHANGE_NAME}}
TASKS_COMPLETE: N/N
VERIFY_RESULT: PASSED | WARNINGS | CRITICAL
ISSUES:
- <list any CRITICAL or WARNING issues, or "none">
BLOCKERS:
- <list anything that stopped progress, or "none">
TESTING REPORT SUMMARY:
- summary of the reports created
VERIFICATION SUMMARY:
- output from verify step as is
```
