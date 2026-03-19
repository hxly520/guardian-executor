# 中断检测与恢复

## 1. 何时判定任务可能中断 / 卡死 / 失联

出现以下任一信号，就要怀疑任务已 stale：

- `updated_at` 长时间未刷新，明显超过该任务正常阶段时长
- `status` 长时间停留在 `running` / `retrying`，但没有新的 checkpoint、snapshot、日志增长
- 关联的 `session_key` / `process_id` 已失效、退出、查不到
- 同一个错误重复出现，多次重试仍没有推进到下一阶段
- 主会话恢复后，发现任务仍显示运行中，但没有新的产出或状态变化

简单判法：

**状态没动 + 日志没长 + 检查点没更新 = 优先按中断或卡死处理。**

## 2. 恢复前先看哪些文件

按这个顺序检查：

1. `memory/task-state.json`
   - 看任务是否还在活跃列表
   - 看 `status`、`updated_at`、`attempt_count`、`last_error`、`next_retry_at`
2. 项目级 `runtime/*.json`
   - 看当前阶段、阶段结果、executor 信息、checkpoint 路径
3. `SESSION_SNAPSHOT.md` 或项目快照
   - 看任务目标、已完成内容、下步计划、blocker
4. logs / command output / process notes
   - 看最后停在哪一步
5. checkpoint 文件
   - 看是否已经有可直接继续消费的中间产物

**恢复优先依赖状态文件，不优先回放整段聊天。**

## 3. 恢复原则

### 先恢复，不先重跑

恢复顺序固定：

1. 尝试继续已有 owner agent / guardian runtime
2. 如果原单元已失效，但状态完整，就新开单元并从 checkpoint / state 继续
3. 只有当状态、checkpoint、日志都不足以恢复时，才考虑从头重跑

### 恢复时至少做这些动作

- 明确上一阶段完成到了哪里
- 标记当前恢复动作是 `resume` 而不是全新执行
- 更新 `attempt_count`、`updated_at`、`last_error` / `last_success`
- 如果更换执行单元，更新 `executor`、`session_key`、`process_id`
- 保留旧错误，不要覆盖掉失败证据

## 4. 对不稳定任务的建议

对外部 API、长构建、抓取、同步、批处理等不稳定任务，建议：

- 使用明确的 `retry_policy`（如 fixed_backoff / exponential_backoff）
- 每跨过一个阶段就写 checkpoint，不要只在最后写一次
- 心跳 / 进度更新时间应和任务粒度匹配；阶段型任务至少在阶段切换时刷新 `updated_at`
- 对容易卡住的任务，记录最近一次成功阶段，而不是只记录“正在运行”
- 连续重复失败达到阈值后，转为 `blocked` 或 `failed`，不要无限重试

## 5. 主会话中断后的恢复规则

如果主会话自己断了、/new 了、或上下文丢失：

1. 先查 `memory/task-state.json`
2. 再查项目快照和 runtime 状态
3. 判断有没有仍可复用的 owner/runtime
4. 有就优先接回原链路；没有再考虑新开执行单元
5. 不要因为主会话失忆，就默认把任务当成全新需求重新开跑

## 6. 何时必须停止自动恢复

出现以下情况时，不要盲目继续自动恢复：

- 状态文件彼此矛盾，无法判断真实阶段
- checkpoint 缺失，且重跑成本高 / 风险高
- 连续多次恢复都卡在同一失败点
- 任务依赖人工确认、外部凭据、或用户决策

这时应把任务标成 `blocked` 或 `failed`，明确说明当前恢复边界。
