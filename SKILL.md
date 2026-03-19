---
name: guardian-executor
description: Guardian-first detached execution workflow for any task that will involve execution, probing, diagnosis, API calls, scripts, service checks, builds, syncs, data jobs, or other failure-prone multi-step work. Use when the main session should first create a guarded execution unit, then let that unit perform all probe / diagnose / API / script / service operations with durable state tracking, verification, completion reporting, and cleanup. Do not probe first in the main session and only then decide whether to detach.
---

# Guardian Executor

把这个 skill 作为 **执行类任务的默认入口**，不是“长任务跑到一半再守护化”的兜底方案。

## 核心规则

**Guardian 要先于探测发生。**

只要用户请求后续会涉及以下任一动作，就应先创建执行单元，再在执行单元内部完成后续工作：

- probe / inspect / diagnose
- API 调用
- 脚本运行
- 服务检查 / 服务控制
- 构建 / 测试 / 同步 / 回填 / 抓取
- 代码或文件修改
- 任何需要重试、恢复、状态追踪的执行型任务

主会话不要先自己 probe 一轮、试一轮、跑一轮，再决定是否守护化。
主会话只负责：**接单、拆解、开执行单元、接收结果、向用户汇报**。

## 不适用场景

以下情况通常可以不触发：

- 单次只读文件查看
- 很短的纯文字澄清
- 不涉及外部执行 / 探测 / API / 脚本 / 服务检查的轻量查询
- 几秒内完成、且没有恢复价值的极小改动

## 固定工作流（低自由度）

按下面顺序执行，不要重排：

### 1）先判定是否属于执行类任务

如果后续会发生执行、探测、诊断、API、脚本、服务、构建、同步、抓取、批处理等动作，直接进入 Guardian Executor 工作流。

### 2）先做任务归属判断（owner-aware routing）

在创建执行单元之前，必须先判断任务属于哪一层，禁止把跨层需求整包丢给“最近看起来能改的人”。

按下面规则硬判断：

1. **UI-only / 展示层任务**
   - 只涉及视觉样式、排版、前端展示文案落位、页面交互、组件布局、展示层映射。
   - 不改变业务语义，不重新定义字段含义，不决定名称标准，不解释策略/因子逻辑。
   - 这类任务可以直接路由给前端 / 页面执行单元。

2. **Domain-owned / 语义归属任务**
   - 只要需求涉及以下任一项，就不能直接交给前端执行单元：
     - 名称标准化 / 命名收敛
     - 业务语义定义
     - 因子解释 / 指标解释 / 策略归因
     - 数据含义、口径、后端产物命名
     - owner 确认、责任域确认
     - “这个东西为什么这样命名 / 怎么挖掘出来 / 基于什么分析”
   - 这类任务必须先路由给对应 owner agent 或领域执行单元，由其产出统一命名、定义、说明、语义边界。
   - 在 owner 产出之前，前端执行单元不得自行拍脑袋定义 display label、解释文案或语义映射。

3. **Mixed cross-layer / 混合跨层任务**
   - 既包含上游语义/逻辑定义，又包含下游页面适配、展示消费、UI 落位时，必须拆单。
   - 固定拆法：
     1. 先创建上游 owner 执行单元，完成命名、定义、解释、归因、口径确认。
     2. 再创建下游前端执行单元，消费上游产物做页面适配与展示。
   - 禁止把混合需求整包交给单一前端子会话，也禁止让前端先做一版“临时显示名”再倒逼上游对齐。

如果无法明确 owner，先补做 owner 判断，再开执行单元；不要默认前端兜底。

### 3）先创建执行单元

优先级固定如下：

1. **已有项目 guardian / runtime task**
   - 项目本身已有守护链时优先复用。
2. **sub-agent 执行单元**
   - 默认选项；适合编码、实现、审计、排查、结构化任务。
3. **background exec**
   - 仅在必须使用宿主后台进程时兜底。

在执行单元创建之前，主会话不要做实质性 probe、diagnose、API 调用或脚本执行。

### 4）在执行单元内初始化状态

执行单元开始后，先写入或更新至少一个可恢复状态位，例如：

- `memory/task-state.json`
- 项目级 `runtime/*.json`
- `SESSION_SNAPSHOT.md`
- 专项 checkpoint 文件

状态结构参考：
- `references/task-state-schema.md`

### 5）在执行单元内完成所有探测与执行

以下动作都应在执行单元内部发生，而不是主会话：

- probe / diagnose
- API 调用
- script / build / test
- service status / start / stop / restart
- 数据抓取 / 回填 / 迁移 / 同步
- 修复、验证、重试

### 7）避免紧轮询

不要为了“看起来在跟进”而频繁轮询。
优先使用：

- 推送式完成事件
- 粗粒度状态检查
- 长超时 poll

### 7）验证后再宣布完成

可接受的验证方式包括：

- build / test 通过
- API 返回 200 / 预期结果
- 服务状态 active
- 目标文件存在且内容正确
- 状态文件显示成功
- 代码任务有真实 commit

### 8）由主会话做最终回报

执行单元完成后，主会话统一向用户回报至少这些内容：

- 做了什么
- 怎么验证的
- commit id 或产出状态
- blocker / 剩余风险

模板见：
- `references/reporting-templates.md`

### 9）一次性任务完成后清理

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
- `references/reporting-templates.md`：启动 / 进度 / 完成 / 失败模板
