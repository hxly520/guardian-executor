---
name: guardian-executor
description: Default detached execution workflow for long-running, multi-step, or failure-prone tasks. Use when code changes, scripts, data syncs, builds, migrations, service operations, or backfills should run in a guarded execution unit with durable status tracking, completion reporting, and cleanup.
homepage: https://github.com/hxly520/guardian-executor
metadata: {"clawdbot":{"emoji":"🛡️"}}
---

# Guardian Executor

把这个 skill 作为 **非瞬时任务的默认执行模式**。

## 什么时候用

当任务满足以下任一特征时，优先触发 `guardian-executor`：

- 任务长度超过一个短交互回合
- 预计会涉及多次工具调用
- 需要修改代码或文件
- 需要运行脚本 / 构建 / 测试 / 同步 / 回填 / 服务控制
- 容易被外部 API / 网络波动打断
- 用户明确要求后台执行、守护执行、执行单元方式
- 任务必须在聊天中断后仍能恢复

以下情况通常 **不要** 触发：

- 单次只读查询
- 很短的纯问答
- 几秒内结束的小改动

## 核心原则

对于匹配的任务，**不要默认绑在当前聊天回合里硬跑**。  
默认切换到守护式执行单元。

推荐优先级：

1. **已有项目 guardian / runtime task**
   - 项目本身已经有守护链时，优先复用它
2. **sub-agent 执行单元**
   - 适合编码、实现、审计、结构化任务
3. **background exec**
   - 仅在必须使用宿主后台进程时兜底

## 标准工作流

### 1）先判断是否要走守护执行

如果任务是长、复杂、易失败、可恢复需求强，就应进入 Guardian Executor 工作流。

### 2）先落状态，再开始执行

在真正执行前，写入或更新至少一个可恢复状态位，例如：

- `memory/task-state.json`
- 项目级 `runtime/*.json`
- `SESSION_SNAPSHOT.md`
- 专项 checkpoint 文件

状态结构参考：
- `references/task-state-schema.md`

### 3）创建执行单元

对编码 / 多步骤实现类工作，优先使用 `sessions_spawn`。  
如果项目已有 guardian 任务，就直接复用 guardian 常驻链。  
只有在必须跑宿主进程时，才使用后台 `exec`。

### 4）避免紧轮询

不要为了“看起来在跟进”而频繁轮询。  
优先使用：

- 推送式完成事件
- 粗粒度状态检查
- 长超时 poll

### 5）必须验证后再回报完成

可接受的验证方式包括：

- build / test 通过
- API 返回 200
- 服务状态 active
- 目标文件存在
- 状态文件显示成功
- 代码任务有真实 commit

### 6）完成回报必须固定化

一次性任务完成时，至少回报：

- 做了什么
- 怎么验证的
- commit id 或产出状态
- blocker / 剩余风险

模板见：
- `references/reporting-templates.md`

### 7）一次性任务完成后要清理

- 一次性任务：从活跃执行单元中剔除，或标记 completed
- 常驻任务：保留，并持续更新状态

## Git 纪律

对代码任务，必须遵守：

- 只 `git add` 明确目标文件
- 不要 `git add .`
- 不要没 commit 就说完成
- 汇报时必须带最终 commit id

## 失败纪律

如果任务失败：

- 保留状态文件
- 记录最后错误
- 记录下一步重试时间或人工下一步
- 如实汇报，不要假装“快好了”

## 参考资料

按需加载：

- `references/task-lifecycle.md`：任务生命周期
- `references/task-state-schema.md`：状态文件结构
- `references/reporting-templates.md`：启动 / 进度 / 完成 / 失败模板
