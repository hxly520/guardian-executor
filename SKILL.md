---
name: guardian-executor
description: Default detached execution workflow for long-running, multi-step, or failure-prone tasks. Use when code changes, scripts, data syncs, builds, migrations, service operations, or backfills should run in a guarded execution unit with durable status tracking, completion reporting, and cleanup.
metadata: {"clawdbot":{"emoji":"🛡️"}}
---

# Guardian Executor

Use this skill as the **default execution mode** for non-trivial work.

## When to use

Use `guardian-executor` when the task is any of the following:

- longer than a short conversational turn
- likely to involve multiple tool calls
- likely to modify code or files
- likely to run scripts / build / test / sync / backfill / restart services
- likely to fail intermittently and need retries
- important enough that status must survive chat interruption
- explicitly requested to run in background / detached / guardian style

Do not use it for quick read-only lookups or tiny one-shot edits.

## Core policy

For matching tasks, do **not** default to chat-bound inline execution.
Default to a guarded execution unit instead.

Preferred execution order:

1. **Existing project guardian / runtime task**
   - Use when the project already has registered recurring tasks or a runtime daemon.
2. **Sub-agent execution unit**
   - Use `sessions_spawn` for coding / structured task execution that should report back when complete.
3. **Background exec**
   - Use only when direct host commands must continue independently and sub-agent execution is not the right fit.

## Required workflow

### 1) Decide if the task is guarded

If the task is long, multi-step, retry-prone, or user-visible as a tracked job, run it through Guardian Executor.

### 2) Create a durable status record

Write or update a task entry in:

- `memory/task-state.json`, or
- a project-local runtime/status file, or
- both, when the work is important

Read `references/task-state-schema.md` for the minimum schema.

### 3) Start the execution unit

For coding and agentic work, prefer `sessions_spawn`.

The kickoff update should tell the user only:

- objective
- execution mode
- where status is tracked
- what completion report will include

Keep kickoff updates short.

### 4) Avoid tight polling

Do **not** burn turns with constant polling.
Prefer:

- push-based child completion events
- coarse status checks only when useful
- long-timeout `process poll` if background exec must be checked

### 5) Verify before claiming done

Before reporting completion, confirm the result with concrete checks such as:

- build/test success
- API response
- service status
- file presence
- log or state evidence

### 6) Report in a fixed completion shape

For one-shot tasks, completion updates should include:

- what changed
- verification performed
- commit id or output status
- blocker status / remaining risks

Use the templates in `references/reporting-templates.md`.

### 7) Clean up

If the execution unit is one-shot:

- mark task state `completed` or remove it from the active registry
- do not leave zombie task entries

If it is recurring:

- keep the entry
- keep status queryable

## Git discipline

For coding tasks:

- stage only explicit target files
- avoid broad `git add .`
- report the final commit id
- do not claim completion without a real commit when the task requires code changes

## Failure discipline

If the job fails:

- keep the state record
- record the last error
- record next retry / manual next step
- report failure honestly instead of silently retrying forever without visibility

## References

Load these only when needed:

- `references/task-lifecycle.md` — detailed execution flow
- `references/task-state-schema.md` — durable status schema
- `references/reporting-templates.md` — kickoff / progress / completion templates
