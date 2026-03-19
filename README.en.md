# Guardian Executor 🛡️

A reusable OpenClaw skill that makes **guardian-first execution** the default for execution-heavy work.

GitHub: <https://github.com/hxly520/guardian-executor>

---

## What problem it solves

A lot of agent work should not stay inside the main chat turn.

Typical examples:

- the task needs probing before action
- external APIs may fail or stall
- scripts, builds, or service checks may take time
- recovery matters if the chat is interrupted
- the final answer must include verification and often a commit id

If all of that happens inline in the main session, the common failure modes are predictable:

- the conversation context gets bloated
- failed probes force the agent to restate context and recover manually
- interruption makes it unclear where the task stopped
- the main session ends up carrying state that should have lived in a durable execution unit

`guardian-executor` pushes that work into the right place from the start.

> **If the request will turn into execution work, create the execution unit first, then do the probing, diagnosis, API calls, scripts, and verification inside that unit.**

---

## Core rule

### Guardian must happen before probing

This is the rule the skill is built around.

Not:

- probe first in the main session
- decide later whether the task is “big enough” to detach

Instead:

- if the request will involve execution, probing, diagnosis, API calls, scripts, or service checks
- create a guardian/sub-agent/background execution unit first
- let that unit perform the actual work

### The main session has a narrow job

The main session should mostly do four things:

1. accept the request
2. break down the work
3. create the execution unit
4. report the result back to the user

The execution unit does the real operational work.

### Guardian-first is not nearest-unit dumping

Guardian-first does **not** mean “open any nearby execution unit and throw the whole request there.”

For cross-layer requests, do owner-aware routing first:

- UI-only work can go directly to a frontend/page execution unit
- naming standardization, business semantics, factor explanation, strategy attribution, data meaning, backend artifact naming, or owner confirmation must go to the responsible owner agent first
- mixed requests must be split into upstream owner work first and downstream presentation work second

See [`references/task-routing.md`](./references/task-routing.md) for the strict routing rules.

---

## When to use it

`guardian-executor` is a strong default for:

- code changes, refactors, and multi-file edits
- builds, tests, and release validation
- sync, migration, backfill, crawl, or ETL-style jobs
- unstable or rate-limited external API work
- service inspection, service control, and troubleshooting
- tasks that need retries, checkpoints, or resumability
- requests where the user explicitly wants background or guarded execution

A practical shortcut:

**If the task will actually do work rather than just read and answer, guardian-first usually fits.**

---

## When not to use it

You usually do not need this skill for:

- one-off file reads
- short clarification replies
- tiny changes that finish in seconds
- lightweight queries with no execution, probing, API, script, or service work

This skill is not about pushing everything into the background.
It is about tightening the default workflow for **execution-type tasks**.

---

## Standard workflow

### 1. Decide whether the request will enter execution mode

If the next steps will involve any of the following:

- probe / inspect / diagnose
- API calls
- script / build / test
- service status / start / stop / restart
- crawl / sync / migrate / backfill
- code or file modification

then do **not** probe inline first.
Go straight into the guardian workflow.

### 2. Create the execution unit first

Recommended priority order:

1. reuse an existing project guardian/runtime task
2. create a sub-agent execution unit
3. fall back to background exec only when host-level execution is required

### 3. Initialize durable state inside the execution unit

Examples:

- `memory/task-state.json`
- project-level `runtime/*.json`
- `SESSION_SNAPSHOT.md`
- checkpoint files

### 4. Do all probing, execution, repair, and verification inside the unit

The probing phase belongs to the execution unit, not to the main chat.

### 5. Report only at meaningful checkpoints

Avoid noisy tight polling.

### 6. Verify before claiming completion

Completion reports should include at least:

- what was done
- how it was verified
- commit id or output status
- blockers or remaining risk

### 7. Clean up one-shot execution units when the work is done

Do not leave fake active state behind.

---

## Does this reduce token usage?

The careful answer is:

**Usually yes for complex or unstable tasks, but not guaranteed for every tiny task.**

### Where it often saves tokens

For multi-step or failure-prone work, guardian-first often reduces:

- main-session context growth
- repeated recovery explanations after interruptions or failed probes
- the need to rebuild state from chat history

### Where it may not save tokens

For very small and near-instant tasks:

- creating an execution unit has its own overhead
- total token usage may be slightly higher than doing the work inline

So the useful takeaway is not “always fewer tokens.”
It is:

> **For complex or unstable execution work, guardian-first is usually more reliable and often more token-efficient in practice. For tiny tasks, do not apply it mechanically.**

---

## Repository contents

- `SKILL.md` — skill definition and strict workflow
- `README.md` — entry page
- `README.zh-CN.md` — Chinese documentation
- `README.en.md` — English documentation
- `references/task-routing.md` — owner-aware routing guidance
- `references/task-lifecycle.md` — lifecycle guidance
- `references/task-state-schema.md` — durable state schema
- `references/reporting-templates.md` — reporting templates

---

## Who this is for

This skill is useful for teams or operators who want agents to:

- detach execution work early
- keep durable, queryable task state
- recover cleanly after failures or interruptions
- report completion with real verification

---

## What it does not try to do

This skill does **not** try to:

- force every task into the background
- maximize autonomy for its own sake
- wrap tiny work in heavy process

Its goal is simpler:

**send execution-type work into the right execution channel from the very beginning.**

---

## License

MIT
