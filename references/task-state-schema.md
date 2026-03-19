# Task State Schema

Use this as the minimum durable shape.

```json
{
  "tasks": [
    {
      "task_id": "task-20260319-001",
      "objective": "Refresh formal fund universe and verify readiness",
      "status": "running",
      "started_at": "2026-03-19T15:00:00+08:00",
      "updated_at": "2026-03-19T15:05:00+08:00",
      "attempt_count": 1,
      "retry_policy": "exponential_backoff",
      "last_error": null,
      "last_success": null,
      "next_retry_at": null,
      "checkpoint_file": "projects/example/runtime/task-20260319-001.json",
      "executor": "subagent",
      "session_key": "agent:main:subagent:xxxx",
      "process_id": null,
      "notes": "Waiting for verification",
      "result_summary": null
    }
  ]
}
```

## Status values

Recommended values:

- `queued`
- `running`
- `retrying`
- `blocked`
- `completed`
- `failed`
- `cancelled`

## Cleanup rule

- one-shot tasks: remove from active registry after successful completion, or mark `completed`
- recurring tasks: keep active and keep `updated_at` fresh

## Queryability rule

A new session should be able to answer:

- what is running
- what failed
- what finished
- where to resume

without needing old chat history.
