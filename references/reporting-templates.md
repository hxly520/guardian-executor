# Reporting Templates

## Kickoff

```text
已开始处理。
- 目标：<objective>
- 方式：guardian-executor / subagent / background exec / project guardian
- 状态文件：<path>
- 完成后我会回：结果、验证、commit、blocker
```

## Mid-progress

```text
进度更新：
- 当前阶段：<stage>
- 已完成：<done>
- 正在执行：<running>
- 如有风险：<risk or none>
```

## Completion

```text
完成了。
- 做了什么：<summary>
- 验证：<checks>
- commit：<commit id or not applicable>
- blocker：<none or details>
```

## Failure

```text
这轮还没完成。
- 当前失败点：<error>
- 已保留状态：<path>
- 下一步：<retry or manual action>
```
