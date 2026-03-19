# Guardian Executor 🛡️

Guardian-first execution workflow for OpenClaw skills and agents.

- 中文说明：[`README.zh-CN.md`](./README.zh-CN.md)
- English: [`README.en.md`](./README.en.md)

## What this repository is

`guardian-executor` turns a runtime habit into a reusable skill:

> **Create a guarded execution unit first, then let that unit do the probing, diagnosis, API calls, scripts, service checks, and verification.**

That means the main conversation should not spend a round probing first and only later decide to detach the task.

## One-line rule

**Guardian should happen before probing.**

If a request will involve execution work, the main session should mainly do four things:

1. accept the task
2. break it down
3. create the execution unit
4. report the final result

The execution unit does the actual work.

## Routing rule

**Guardian-first does not mean “throw everything to the nearest execution unit.”**

For cross-layer requests, do owner-aware routing first:

- UI-only work can go directly to a frontend/page execution unit
- naming, semantics, factor explanation, strategy attribution, data meaning, backend artifact naming, or owner confirmation must go to the responsible owner agent first
- mixed requests must be split: upstream owner work first, downstream presentation work second

Detailed routing guidance lives in [`references/task-routing.md`](./references/task-routing.md).

## Why this exists

Inline execution is fine for tiny read-only questions.
It is much worse for work that can hang, fail halfway, need retries, or lose context if the chat is interrupted.

This skill makes the safer path the default path.

## Token cost: the honest answer

Does guardian-first reduce token usage?

**Usually yes for complex or unstable work, but not always for tiny work.**

In plain language:

- For multi-step, failure-prone, or long-running tasks, guardian-first usually reduces **main-session token bloat**.
- It also reduces the cost of **recovery**, because status, progress, and checkpoints already exist when something fails or the chat gets interrupted.
- But for very small tasks, creating a guarded execution unit can add overhead, so it does **not guarantee the absolute lowest total token count every time**.

So the practical rule is:

- **Use guardian-first by default for execution work.**
- **Do not force it onto tiny, read-only, near-instant tasks.**

## Repository contents

- `SKILL.md` — skill definition and strict workflow
- `README.zh-CN.md` — Chinese public documentation
- `README.en.md` — English public documentation
- `references/task-routing.md` — owner-aware routing guidance
- `references/task-lifecycle.md` — lifecycle guidance
- `references/task-state-schema.md` — durable state schema
- `references/reporting-templates.md` — reporting templates

## Repository

- GitHub: <https://github.com/hxly520/guardian-executor>
- License: MIT
