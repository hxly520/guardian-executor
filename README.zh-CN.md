# Guardian Executor 🛡️

把“守护式执行”做成一个可复用、可公开分享的 OpenClaw Skill。

GitHub 仓库：<https://github.com/hxly520/guardian-executor>

---

## 它解决什么问题

很多执行型任务本质上并不适合直接绑在主会话里做。

比如：

- 需要先探测、再判断、再执行
- 可能卡在外部 API、网络、脚本、构建、服务状态
- 需要跑比较久
- 中途失败后需要重试
- 聊天中断后还得能恢复
- 完成时还要带验证和 commit id

如果这些动作先在主会话里做，常见后果就是：

- 聊天上下文越来越肿
- 探测失败后要重新解释前情
- 一旦中断，很难知道任务做到哪一步了
- 主会话被迫承担本该由后台执行单元承担的脏活和状态管理

`guardian-executor` 的做法很直接：

> **只要后续会进入执行态，就先拉起执行单元，再在执行单元里做探测、诊断、API 调用、脚本运行和验证。**

---

## 核心规则

### Guardian 要先于探测发生

这是这个 skill 最核心的规则。

不是“先 probe 一轮，发现复杂了，再守护化”。
而是：

- 只要请求后续会涉及执行 / 探测 / 诊断 / API / 脚本 / 服务检查
- 就应该先创建 guardian / sub-agent / background exec 这类执行单元
- 然后由执行单元内部完成后续动作

### 主会话只做四件事

主会话主要负责：

1. 接收任务
2. 做必要拆解
3. 判断执行方式并选择执行通道
4. 汇报最终结果

真正的执行工作应留在执行单元内完成。

### guardian-first 不等于就近乱投

guardian-first 的意思不是“反正先开个执行单元，然后把需求整包丢给最近的前端或实现单元”。
也不是“凡事都先新开一个子任务，再说”。

如果请求是跨层任务，必须先做 owner-aware routing，再决定执行方式：

- 纯 UI / 展示层任务，可以直接交给前端 / 页面 owner
- 只要涉及命名标准化、业务语义、因子解释、策略归因、数据含义、后端产物命名、owner 确认，就必须先交给对应 owner agent
- 如果负责该任务的 owner agent 或 guardian runtime 已存在，优先直调，不要机械新开 generic 子任务
- 如果没有可复用链路，且任务会进入执行态，再新开专用执行单元
- 只有纯读取 / 纯澄清 / 无恢复价值的极小动作，才允许 inline
- 混合需求必须拆成上游 owner 单元 + 下游展示单元，不能整包交给一个前端子会话

细则放在：[`references/task-routing.md`](./references/task-routing.md)

### 恢复优先，不靠重放聊天补命

这个 skill 现在还把“中断检测、恢复导向、持续学习”一起固化了：

- 用 task-state、runtime、checkpoint、snapshot、logs 判断任务是否 stale
- 恢复时优先从状态继续，而不是重跑整段聊天上下文
- 当发生误路由、误判执行方式、恢复失败、重复重试时，要把 learnings 回写成规则

细节下沉到：

- [`references/interruption-and-recovery.md`](./references/interruption-and-recovery.md)
- [`references/learning-loop.md`](./references/learning-loop.md)

---

## 什么时候适合用

以下情况，适合默认启用 `guardian-executor`：

- 代码修改、重构、批量编辑
- 构建、测试、发布前验证
- 数据同步、回填、抓取、迁移
- 调外部 API，且结果不稳定或可能限流
- 服务状态检查、服务拉起、重启、运行诊断
- 需要状态落盘、可恢复、可重试的任务
- 用户明确要求后台执行、守护执行、执行单元

一句话判断：

**只要任务后面会“真的动起来”，而不是只读、只答，就应该优先走 guardian-first。**

---

## 什么时候不适合用

以下情况通常不必启用：

- 单次读取文件
- 很短的文字澄清
- 几秒内完成的极小改动
- 完全不涉及探测 / 执行 / API / 脚本 / 服务的轻量查询

这个 skill 不是为了把所有任务都后台化，而是为了把**执行型任务**的默认流程收紧、做稳。

---

## 标准工作流

### 1. 先判断后续是否会进入执行态

如果后面会涉及：

- probe / inspect / diagnose
- API 调用
- script / build / test
- service status / start / stop / restart
- crawl / sync / migrate / backfill
- 代码或文件修改

那就不在主会话里先跑一轮，而是直接进入 guardian 流程。

### 2. 先选执行方式

固定只允许三类方式：

1. 直接调度已有 owner agent / guardian runtime
2. 新开专用执行单元（优先 sub-agent，必要时 background exec）
3. 极轻量 inline（仅限纯读取 / 纯澄清 / 无恢复价值微动作）

只要拿不准，就不要 inline。

### 3. 在执行单元里写状态

执行单元应尽早把状态写到可恢复位置，例如：

- `memory/task-state.json`
- 项目级 `runtime/*.json`
- `SESSION_SNAPSHOT.md`
- checkpoint 文件

### 4. 在执行单元里完成探测、执行、修复、验证

探测不是主会话的活，执行单元才是。

### 5. 只在关键节点汇报

避免密集轮询和刷屏式状态同步。

### 6. 先验证，再宣布完成

完成回报至少要有：

- 做了什么
- 怎么验证的
- commit id 或产出状态
- blocker / 剩余风险

### 7. 一次性任务完成后清理执行单元

任务结束就结束，不要留下假活跃状态。

---

## 会不会更省 token？

谨慎说，**通常会更省主会话 token，但不保证所有小任务的总 token 一定更低。**

更通俗地说：

### 更可能省的地方

对于复杂任务、不稳定任务、会失败重来的任务，guardian-first 往往会：

- 减少主会话上下文膨胀
- 减少“因为中断/报错而重新解释背景”的重复成本
- 减少恢复任务时重新探测、重新组织上下文的 token 消耗

### 不一定省的地方

如果任务非常小、非常快、几乎不会失败：

- 创建执行单元本身也有开销
- 这时总 token 不一定比直接 inline 更低

所以更准确的结论不是“永远更省”，而是：

> **对于复杂或不稳定的执行型任务，guardian-first 通常更稳，也通常更省；对于极小任务，不要机械套用。**

---

## 这个仓库里有什么

- `SKILL.md`：Skill 正文，定义触发描述与低自由度执行流程
- `README.md`：入口页
- `README.zh-CN.md`：中文说明
- `README.en.md`：英文说明
- `references/task-routing.md`：任务归属判断与 owner-aware routing
- `references/task-lifecycle.md`：任务生命周期
- `references/task-state-schema.md`：状态结构建议
- `references/interruption-and-recovery.md`：中断检测与恢复规则
- `references/learning-loop.md`：执行后复盘与规则学习闭环
- `references/reporting-templates.md`：启动 / 进度 / 完成 / 失败模板

---

## 适合谁

这个 skill 适合：

- 想让 agent 默认先守护化，再执行的团队/个人
- 经常做多步骤工程任务、排障、同步、构建的人
- 需要任务可恢复、可验证、可回报的人
- 不想把主会话变成“又探测又执行又记状态”的一锅粥的人

---

## 不追求什么

这个 skill 不追求：

- 把所有事情都后台化
- 让 agent 无限自治
- 用复杂框架包装极小任务

它追求的是一件更朴素的事：

**让执行型任务从一开始就走正确的执行通道。**

---

## License

MIT
