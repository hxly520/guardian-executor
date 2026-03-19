# Guardian Executor 🛡️

> OpenClaw 的默认守护式执行工作流插件。  
> 适合复杂任务、长任务、易失败任务、需要后台持续执行与完成回报的场景。

## 它是干什么的

`guardian-executor` 的目标很直接：

把原本容易绑定在当前聊天回合、容易超时、容易中断、容易丢状态的任务，统一转换成一种更稳的执行方式：

- 创建执行单元
- 后台持续执行
- 状态落盘可恢复
- 避免紧轮询占满上下文
- 完成后返回结果、验证、commit、blocker
- 一次性任务完成后自动清理执行单元

你可以把它理解成：

**“把 guardian 式后台执行，抽成一个通用、可复用、可默认启用的 Agent Skill。”**

---

## 适用场景

推荐在这些场景默认启用：

- 多步骤代码改造
- 长时间构建 / 测试 / 回归验证
- 数据采集 / 回填 / ETL / 抓取
- 服务拉起 / daemon / runtime 编排
- 容易失败、需要重试的外部 API 任务
- 需要在完成后给出 commit 和结果摘要的任务
- 不能依赖当前聊天回合一直不断开的任务

不建议用于：

- 单次文件读取
- 几秒内完成的简单查询
- 纯问答型回复
- 极小的一步式修改

---

## 核心能力

### 1. 自动识别是否该走守护执行
当任务满足“长 / 重 / 易失败 / 易中断”这些特征时，不再默认绑定当前聊天回合，而是切换到执行单元。

### 2. 强制状态可查询
任务不只存在于聊天里，还要落到文件或运行态里，例如：

- `memory/task-state.json`
- 项目级 `runtime/*.json`
- `SESSION_SNAPSHOT.md`
- 专项 checkpoint 文件

### 3. 先验证，再报完成
这个插件要求：

- 不能凭感觉说“做完了”
- 必须带验证结果
- 代码任务必须带 commit id
- 失败时必须保留状态和下一步建议

### 4. 一次性任务完成后自动清理
- 一次性任务：完成就从活跃执行单元中移除
- 常驻任务：继续保留并持续可查询

---

## 推荐执行顺序

插件默认遵循下面这套优先级：

1. **项目已有 guardian / runtime task**
   - 如果项目已经注册了守护任务，优先复用
2. **sub-agent 执行单元**
   - 适合代码实现、结构化分析、明确边界的多步骤任务
3. **background exec**
   - 仅在确实需要宿主级后台进程时使用

---

## 标准完成回报

这个插件要求完成时至少汇报：

- 做了什么
- 怎么验证的
- commit id / 服务状态 / 输出结果
- blocker 是否存在

推荐回报形态：

```text
完成了。
- 做了什么：<summary>
- 验证：<checks>
- commit：<commit id>
- blocker：<none or details>
```

---

## 仓库结构

```text
guardian-executor/
├── README.md
├── LICENSE
├── .gitignore
├── _meta.json
├── SKILL.md
├── .clawhub/
│   └── origin.json
└── references/
    ├── task-lifecycle.md
    ├── task-state-schema.md
    └── reporting-templates.md
```

---

## 安装方式

### 方式 1：作为本地 workspace skill 使用

放到：

```bash
~/.openclaw/workspace/skills/guardian-executor
```

### 方式 2：作为独立项目引用

将本仓库克隆到本地，然后在你的 OpenClaw workspace / skills 目录中链接或复制使用。

---

## 推荐搭配的 workspace 规则

建议在 `AGENTS.md` 中配套加入类似规则：

- 非瞬时执行型任务默认套用 `guardian-executor`
- 纯读取、轻量问答可保留 inline 执行
- 完成时必须返回验证结果与 commit

---

## 文档导航

- `SKILL.md`：技能本体说明与触发策略
- `references/task-lifecycle.md`：完整任务生命周期
- `references/task-state-schema.md`：状态文件推荐结构
- `references/reporting-templates.md`：启动 / 进度 / 完成 / 失败回报模板

---

## 设计原则

这个插件坚持 4 条底线：

1. **长任务不绑死当前对话**
2. **状态必须可恢复**
3. **完成必须可验证**
4. **失败必须诚实可追踪**

---

## 当前定位

`guardian-executor` 不是项目内某一个 guardian 的替代品，
而是把“guardian 式执行模式”抽象成一个通用技能。

也就是说：

- guardian 是项目运行时底座
- `guardian-executor` 是任务执行工作流的统一封装

---

## 仓库信息

- GitHub: <https://github.com/hxly520/guardian-executor>
- 默认分支：`main`

如果你希望后续把它继续包装成可发布到更多 OpenClaw / ClawdHub 场景的版本，可以在这个基础上继续扩展元信息和发布流程。
