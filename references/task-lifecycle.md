# Task Lifecycle

## 1. Intake

Classify the request:

- **inline**: quick read-only, small answer, trivial edit
- **guardian-executor**: long, multi-step, retry-prone, or background-worthy

## 2. State creation

Before real work starts, create a durable task record.

Minimum fields:

- `task_id`
- `objective`
- `status`
- `started_at`
- `updated_at`
- `attempt_count`
- `retry_policy`
- `last_error`
- `last_success`
- `next_retry_at`
- `checkpoint_file`
- `executor`
- `session_key` or `process_id` if available

## 3. Execution choice

### Prefer project guardian/runtime
Use when the repo already has registered long-running tasks.

### Prefer sub-agent session
Use for coding, audits, implementation, or other bounded agentic tasks.

### Prefer background exec
Use when a literal shell process must survive the current turn.

## 4. Progress handling

- do not spam progress
- only send user-visible updates at meaningful milestones
- keep detailed state in files, not in chat only

## 5. Verification

Examples:

- build passes
- endpoint returns 200
- service is active
- task state shows success
- required files exist
- repo has a real commit

## 6. Completion

For one-shot jobs, report:

- final outcome
- concrete verification
- commit id / output status
- blockers or none

Then mark the task completed or remove it from the active queue.

## 7. Retry behavior

If upstream or runtime errors occur:

- preserve the task record
- increment attempt count
- set `next_retry_at`
- record a checkpoint or partial result when available

Never hide that the task is still unresolved.
