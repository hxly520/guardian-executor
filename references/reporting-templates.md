# 回报模板

## 启动回报

```text
已开始处理。
- 目标：<objective>
- 方式：guardian-executor / subagent / background exec / project guardian
- 状态文件：<path>
- 完成后我会回：结果、验证、commit、blocker
```

## 中途进度

```text
进度更新：
- 当前阶段：<stage>
- 已完成：<done>
- 正在执行：<running>
- 风险：<risk or none>
```

## 完成回报

```text
完成了。
- 做了什么：<summary>
- 验证：<checks>
- commit：<commit id or not applicable>
- blocker：<none or details>
```

## 失败回报

```text
这轮还没完成。
- 当前失败点：<error>
- 已保留状态：<path>
- 下一步：<retry or manual action>
```
