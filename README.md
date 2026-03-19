# Guardian Executor

> Default detached execution workflow for OpenClaw long tasks.

`guardian-executor` is a reusable OpenClaw skill for **complex, long-running, or failure-prone work** that should not stay bound to the current chat turn.

It standardizes one execution style:

- create a guarded execution unit
- persist task state
- run in the background when appropriate
- avoid tight polling loops
- report completion with **result + verification + commit id + blockers**
- clean up one-shot execution units after success

## Why this exists

Normal conversational turns are bad at handling:

- long coding sessions
- data backfills / syncs
- builds and migrations
- flaky upstream APIs
- work that needs retries / checkpoints
- situations where the user wants "start it now, tell me when it's done"

Guardian Executor turns those into a repeatable workflow.

## What it does

- **Classifies tasks** into instant vs guarded execution
- **Uses execution units by default** for non-trivial work
- **Tracks status** in durable state files
- **Prefers completion push-backs** over chat-bound waiting
- **Requires explicit verification** before reporting done
- **Requires explicit commit reporting** for coding tasks

## Default trigger cases

Use `guardian-executor` when the task includes any of these:

- multi-step coding work
- scripts / builds / tests / migrations
- data sync / scraping / backfill / ETL
- service restart / daemon / runtime orchestration
- work expected to take more than a short interactive turn
- work that must survive chat interruption
- user asks for background / detached / guardian-style execution

Do **not** use it for:

- one-off file reads
- quick lookups
- short pure-answer requests
- tiny edits that finish in one simple turn

## Repo structure

```text
guardian-executor/
├── README.md
├── LICENSE
├── .gitignore
├── _meta.json
├── SKILL.md
└── references/
    ├── task-lifecycle.md
    ├── task-state-schema.md
    └── reporting-templates.md
```

## Install

### Local workspace skill

Put this folder under your OpenClaw workspace `skills/` directory:

```bash
~/.openclaw/workspace/skills/guardian-executor
```

### Suggested workspace policy

Pair this skill with a workspace rule such as:

- long tasks default to `guardian-executor`
- short read-only work may stay inline

## Core contract

When this skill is used, the assistant should:

1. open a guarded execution unit
2. persist status to a durable file
3. keep user updates concise
4. avoid fake completion
5. return with:
   - what was done
   - how it was verified
   - commit id / service status / blocker status
6. clean up one-shot execution units after completion

## Notes

This project is intentionally lightweight.
The skill instructions live in `SKILL.md`; detailed operating references live in `references/`.
