---
name: guardian-executor
description: Guardian-first execution workflow for requests that will involve execution, probing, diagnosis, API calls, scripts, service checks, builds, syncs, data jobs, or other failure-prone multi-step work. Use when the main session must choose the execution mode first, then route work into the right execution channel with durable state, interruption detection, recovery, verification, and cleanup. Supports three dispatch modes: reuse an existing owner agent / guardian runtime, open a dedicated execution unit, or allow extremely light inline handling only for pure read / pure clarification / no-recovery-value micro-actions. Do not probe first in the main session and only later decide whether to detach.
---

# Guardian Executor

把这个 skill 作为 **执行类任务的默认入口**。它不仅要求 guardian 要先于探测发生，也要求主会话先做 **执行方式判断**，再把任务送进正确通道。

## 核心规则

**先判断执行方式，再开始任何实质性执行。**

只要用户请求后续会涉及以下任一动作，就不要在主会话里硬跑：

- probe / inspect / diagnose
- API 调用
- 脚本运行
- 服务检查 / 服务控制
- 构建 / 测试 / 同步 / 回填 / 抓取
- 代码或文件修改
- 任何需要重试、恢复、状态追踪的执行型任务

主会话只负责：**接单、判断归属、选择执行方式、创建或直调执行单元、接收结果、向用户汇报**。

## 三种执行方式（固定判法）

只允许以下三类方式；按顺序判断，尽量降低自由度。

### 方式 1：直接调度已有 owner agent / guardian runtime

这是 **默认优先级最高** 的方式。满足任一信号就优先直调，不要机械新开 generic 子任务：

- 已经存在明确 owner agent、项目 guardian、runtime task、长期执行单元
- 任务是已有任务的继续、补充、追问、恢复、复查、重试、收尾
- 已有状态文件、checkpoint、日志、snapshot 足以继续推进
- 工作依赖该 owner 单元已有上下文、权限、运行资源、未完成产物
- 同一项目下此前已经建立了稳定的 guardian 链路

**强制规则：**
- 能复用已有 owner / runtime，就先复用。
- 不要把“继续已有任务”误拆成全新 generic 子任务。
- 不要让下游前端或 generic coder 代替领域 owner 做语义决策。

### 方式 2：新开专用执行单元（sub-agent / background exec）

当不存在可复用 owner/runtime，或任务需要独立隔离、独立状态、独立后台执行时，创建新单元。

优先级固定：
1. **sub-agent 执行单元**：默认选项，适合编码、实现、审计、排查、结构化多步骤工作。
2. **background exec**：仅在必须依赖宿主后台进程、长驻命令、TTY/进程级控制时使用。

新开专用单元的触发信号：

- 这是一个新的执行型任务，而不是已有 guardian 的续作
- 后续需要脚本、API、构建、测试、服务操作、批处理、抓取、修改文件
- 任务可能超过一个阶段，可能失败，需要 checkpoint / retry / recovery
- 需要与主会话隔离，避免主会话上下文膨胀
- 需要真实落盘状态，供后续恢复或查询

**强制规则：**
- 只要后续会进入执行态，就不要把任务留在主会话里硬跑。
- 主会话不要先试一轮 probe / API / script，再决定要不要新开单元。

### 方式 3：极轻量 inline

inline 只允许用于 **纯读取 / 纯澄清 / 无恢复价值的极小动作**。同时满足以下条件才允许：

- 纯读取文件、纯读取已有上下文、纯短答、纯澄清
- 不调用外部 API，不跑脚本，不做 probe，不控服务，不改文件
- 或者只是一个几秒内完成、失败也没有恢复价值的微小动作
- 不需要 checkpoint、重试、后台持续状态、跨回合恢复

任一条件不满足，就不要 inline。

**禁止把以下任务留在 inline：**
- 需要执行、探测、诊断、脚本、API、构建、测试、抓取、同步
- 需要修改代码或文件
- 需要状态追踪、恢复、重试
- 明显会跨多个阶段

## 不适用场景

以下情况通常可以不触发 guardian-executor：

- 单次只读文件查看
- 很短的纯文字澄清
- 不涉及外部执行 / 探测 / API / 脚本 / 服务检查的轻量查询
- 没有恢复价值的极小动作

## 固定工作流（低自由度）

按下面顺序执行，不要重排。

### 1）先判定是否属于执行类任务

如果后续会发生执行、探测、诊断、API、脚本、服务、构建、同步、抓取、批处理、修改等动作，直接进入 guardian-executor 工作流；不要先在主会话里试跑。

### 2）先做任务归属判断（owner-aware routing）

在选择执行方式之前，先判断任务属于哪一层，禁止把跨层需求整包丢给“最近看起来能改的人”。

按下面规则硬判断：

1. **UI-only / 展示层任务**
   - 只涉及视觉样式、排版、前端展示文案落位、页面交互、组件布局、展示层映射。
   - 不改变业务语义，不重新定义字段含义，不决定名称标准，不解释策略/因子逻辑。
   - 这类任务可以直接路由给前端 / 页面 owner；若已有该 owner runtime，优先直调。

2. **Domain-owned / 语义归属任务**
   - 只要需求涉及以下任一项，就不能直接交给前端执行单元：
     - 名称标准化 / 命名收敛
     - 业务语义定义
     - 因子解释 / 指标解释 / 策略归因
     - 数据含义、口径、后端产物命名
     - owner 确认、责任域确认
     - “这个东西为什么这样命名 / 怎么挖掘出来 / 基于什么分析”
   - 这类任务必须先路由给对应 owner agent 或领域执行单元。

3. **Mixed cross-layer / 混合跨层任务**
   - 既包含上游语义/逻辑定义，又包含下游页面适配、展示消费、UI 落位时，必须拆单。
   - 固定拆法：
     1. 先让上游 owner 单元完成命名、定义、解释、归因、口径确认。
     2. 再让下游展示单元消费上游产物做页面适配与展示。
   - 禁止把混合需求整包交给单一前端子会话。

细则见：`references/task-routing.md`

### 3）再做执行方式判断（dispatch-mode selection）

按这个顺序判定，不要跳步：

1. **是否存在可复用 owner / guardian runtime？**
   - 是：直接调度已有 owner agent / guardian runtime。
2. **否则，这是否是新的执行型任务？**
   - 是：新开专用执行单元，优先 sub-agent，必要时 background exec。
3. **否则，是否同时满足极轻量 inline 条件？**
   - 是：允许 inline。
4. **只要拿不准，就不要 inline。**
   - 回到方式 1 或方式 2。

### 4）在执行单元内初始化状态

执行单元开始后，先写入或更新至少一个可恢复状态位，例如：

- `memory/task-state.json`
- 项目级 `runtime/*.json`
- `SESSION_SNAPSHOT.md`
- 专项 checkpoint 文件

状态结构参考：
- `references/task-state-schema.md`
- `references/task-lifecycle.md`

### 5）在执行单元内完成所有探测与执行

以下动作都应在执行单元内部发生，而不是主会话：

- probe / diagnose
- API 调用
- script / build / test
- service status / start / stop / restart
- 数据抓取 / 回填 / 迁移 / 同步
- 修复、验证、重试

### 6）做中断检测与 stale detection

执行单元推进过程中，定期判断任务是否可能已经中断、卡死、失联或状态陈旧。

最少检查：

- `updated_at` 是否长时间未刷新
- 当前 `status` 是否长期停留在 `running` / `retrying` 但没有新 checkpoint
- runtime / snapshot / logs 是否停止增长
- process_id / session_key 对应执行单元是否还活着（若可查）
- 是否发生重复报错、重复重试、同阶段循环

恢复与判定细则见：`references/interruption-and-recovery.md`

### 7）优先从状态恢复，不要重跑整段聊天上下文

如果任务中断、主会话断开、执行单元退出或需要续跑：

1. 先看状态文件
2. 再看 checkpoint / runtime / snapshot / logs
3. 判断是继续已有 owner/runtime，还是在新单元里从 checkpoint 恢复
4. 只有在状态不可用时，才回退到重新理解聊天上下文

### 8）避免紧轮询

不要为了“看起来在跟进”而频繁轮询。优先使用：

- 推送式完成事件
- 粗粒度状态检查
- 长超时 poll

### 9）验证后再宣布完成

可接受的验证方式包括：

- build / test 通过
- API 返回 200 / 预期结果
- 服务状态 active
- 目标文件存在且内容正确
- 状态文件显示成功
- 代码任务有真实 commit

### 10）执行后做简短复盘与学习回写

当发生以下情况时，记录为 learnings，并反向优化本 skill：

- 误路由：本应找 owner，却错投 generic 单元
- 误判执行方式：本应直调 owner 却新开 generic；本应守护化却留在 inline
- 恢复失败：状态不够、checkpoint 无法续跑、只能重跑上下文
- 重复重试：同类错误多次出现，说明规则或状态粒度不够

学习闭环细则见：`references/learning-loop.md`

### 11）由主会话做最终回报

执行单元完成后，主会话统一向用户回报至少这些内容：

- 做了什么
- 怎么验证的
- commit id 或产出状态
- blocker / 剩余风险

模板见：
- `references/reporting-templates.md`

### 12）一次性任务完成后清理

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

- `references/task-routing.md`：任务归属判断与 owner-aware routing
- `references/task-lifecycle.md`：任务生命周期
- `references/task-state-schema.md`：状态文件结构
- `references/interruption-and-recovery.md`：中断检测、陈旧状态判断、恢复策略
- `references/learning-loop.md`：执行后复盘、误判回写、规则持续优化
- `references/reporting-templates.md`：启动 / 进度 / 完成 / 失败模板
