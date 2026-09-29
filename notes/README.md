# Codex 学习笔记

这个分支（`learning`）是从 `main` 切出的长期学习分支，用来沉淀阅读 [openai/codex](https://github.com/openai/codex) 源码的笔记。

所有代码引用都经过实读验证，格式为 `路径:行号`。行号对应 fork 时的上游版本，上游改动后可能漂移——引用片段本身是逐字抄的，可用关键词重新定位。

## 目录

| 文件 | 内容 |
|---|---|
| [loops-agents-sandbox-go.md](loops-agents-sandbox-go.md) | 三条链串起来：收件循环/双层回合、spawn 子会话、exec 时启动工具沙箱，Go 结构示意 |
| [multi-agent.md](multi-agent.md) | 多智能体：Session/task 拓扑、spawn/配置继承、V1/V2 终态通知、两道限流闸、Go 视角的坑 |
| [skills-vs-tools.md](skills-vs-tools.md) | Skill 与 Tool 的本质区别：渐进披露、两条注入路径、三种 role、目录预算 |
| [fault-tolerance.md](fault-tolerance.md) | 容错：退避与抖动、传输降级、错误分类、panic 传播链、上下文压缩 |
| [turn-loop.md](turn-loop.md) | 两层循环与审批挂起：外层插话入口、内层 ReAct、oneshot 协作式挂起、五张 waiter 表的退场路径、Go 翻译 |
| [exec-command-chain.md](exec-command-chain.md) | exec_command 调用链：审批在 spawn 前、审批≠脱沙箱、三态审批需求、SandboxDenied 升级重试、平台变换与子进程诞生点 |
| [session-registry.md](session-registry.md) | 会话注册表：map 何时写、resume 三路径（SQLite 只是索引）、启动零加载、三层存储 |
| [mcp-architecture.md](mcp-architecture.md) | MCP 拓扑：rmcp 钉版 3.2.0、Legacy/V20260728 双模式、仅 Stdio+StreamableHttp、TUI 反向 server（127.0.0.1:0 + AUTHORIZATION） |
| [sandbox-permissions.md](sandbox-permissions.md) | 沙箱权限：legacy 预设与命名 profile 的区别、读权限例外、工具子进程的沙箱选择、Windows ReadOnly 降级 |
| [compaction.md](compaction.md) | 上下文压缩全解：触发地图、三路分发、90%/95% 双阈值、AI 摘要算法、模板全文 |
| [interview-qa.md](interview-qa.md) | Agent 开发面试题对照本项目源码作答（含来源与可信度说明） |

## 可视化

| 文件 | 内容 |
|---|---|
| [walkthrough-codex-architecture.html](../walkthrough-codex-architecture.html) | 整体架构图：四个前端 → SQ/EQ 协议 → 会话/回合循环 → 模型与工具 → 审批/沙箱 |
| [walkthrough-codex-model-loop.html](../walkthrough-codex-model-loop.html) | 模型输出分诊：流式 `ResponseItem` 如何经 `build_tool_call` 分流成三条路 |

浏览器直接打开即可，节点可点击查看代码。

## 阅读代码的入口

想自己顺着读，这几个文件是主干：

```
codex-rs/protocol/src/protocol.rs          Op / EventMsg 定义 —— SQ/EQ 协议本体
codex-rs/core/src/session/turn.rs          run_turn：回合循环
codex-rs/core/src/stream_events_utils.rs   模型输出的分诊点
codex-rs/core/src/tools/router.rs          build_tool_call：唯一的分类器
codex-rs/core/src/tools/parallel.rs        工具执行与并发门控
codex-rs/core/src/agent/control.rs         多智能体控制面
```

## 约定

- 每个结论都尽量给到 `文件:行号`，方便回查。
- 引用代码一律逐字抄写，不做改写或截断重排。
- 指出设计缺口时标明是「观察」而非「上游结论」。
