# 状态文件结构建议

推荐最低结构如下：

```json
{
  "tasks": [
    {
      "task_id": "task-20260319-001",
      "objective": "刷新正式基金池并校验 readiness",
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
      "notes": "等待验证完成",
      "result_summary": null
    }
  ]
}
```

## 推荐状态枚举

- `queued`
- `running`
- `retrying`
- `blocked`
- `completed`
- `failed`
- `cancelled`

## 清理规则

- 一次性任务：成功后移出活跃队列，或标记 `completed`
- 常驻任务：保留，并持续刷新 `updated_at`

## 可查询性要求

新的会话必须能够只依赖状态文件回答这些问题：

- 现在有哪些任务在跑
- 哪些任务失败了
- 哪些任务完成了
- 从哪里继续恢复

不能依赖旧聊天历史才能还原任务状态。
