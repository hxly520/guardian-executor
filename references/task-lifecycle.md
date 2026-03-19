# 任务生命周期

## 1. 任务进入

先把任务分成两类：

- **inline**：只读、很短、一步完成
- **guardian-executor**：长、多步骤、易失败、需要恢复能力

## 2. 状态初始化

正式执行前，先写一个可恢复状态记录。

最低字段建议包括：

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
- `session_key` 或 `process_id`

## 3. 选择执行模式

### 优先：项目已有 guardian / runtime
适合项目已经有注册好的守护链。

### 其次：sub-agent 执行单元
适合编码、实现、审计、结构化多步骤任务。

### 最后：background exec
适合必须以宿主后台进程方式持续运行的任务。

## 4. 进度处理

- 不要刷屏式汇报
- 只有在真正跨过阶段点时再汇报
- 详细状态尽量写入文件，不要只留在聊天里

## 5. 验证

常见验证包括：

- build 通过
- test 通过
- 接口返回 200
- 服务 active
- 目标文件存在
- git commit 存在

## 6. 完成

一次性任务完成时，应回报：

- 最终结果
- 具体验证
- commit id / 服务状态 / 输出状态
- blocker 或明确无 blocker

然后把任务从活跃执行单元中清理掉，或标记为 completed。

## 7. 重试与恢复

如果任务失败：

- 保留状态记录
- attempt_count +1
- 写入 `next_retry_at`
- 如果有阶段成果，写入 checkpoint

不要隐藏“任务尚未完成”这一事实。
