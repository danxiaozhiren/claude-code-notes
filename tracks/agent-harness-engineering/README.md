# Agent Harness 工程学习系列

> 社区项目核查日期：2026-06-10
> 写作仓库：`danxiaozhiren/claude-code-notes`
> 学习对象：`shareAI-lab/learn-claude-code`
> 系列定位：通过 `learn-claude-code` 社区教学代码理解 Claude Code-like Agent Harness 的通用工程机制。
> 当前状态：8 篇初稿已完成，已补统一索引、术语表、实验手册和收尾审计报告。

## 边界说明

`shareAI-lab/learn-claude-code` 是社区教学项目，不是 Anthropic Claude Code 官方源码，也不代表 Claude Code 官方内部实现。

本系列写作时统一区分四类信息：

| 类型 | 含义 | 使用方式 |
| --- | --- | --- |
| 官方事实 | 来自 Anthropic / Claude Code 官方文档或官方仓库 | 只用于确认 Claude Code 的公开能力和概念边界 |
| 社区教学实现 | 来自 `shareAI-lab/learn-claude-code` 的 README 与 `code.py` | 作为可运行教学样本分析 |
| 作者推导 | 基于官方事实、教学代码和通用工程经验形成的解释 | 明确标注为推导，不写成官方结论 |
| 动手观察 | 读者可在本地复现实验中看到的现象 | 用输入、预期观察、验证标准组织 |

## 这组文章解决什么问题

这组文章不是 Claude Code 官方源码解读，也不是 `learn-claude-code` 的逐章摘要。

它关心的是一个更通用的问题：

```text
一个 LLM 怎么被工程化成能持续行动、受控执行、可恢复、可协作、可扩展的 Agent Harness？
```

八篇文章按运行时能力逐层展开：

| 层级 | 核心能力 | 对应文章 |
| --- | --- | --- |
| 行动循环 | model -> tool_use -> handler -> tool_result -> model | 1 |
| 工具运行时 | 工具 schema、handler、路径安全、动态工具池 | 2、8 |
| 控制面 | 权限、hooks、审计、确定性拦截 | 3、4 |
| 上下文面 | Todo、Skill、Compact、Memory、System Prompt | 5 |
| 稳定性 | retry、reactive compact、任务系统、后台任务、定时调度 | 6 |
| 协作面 | Subagent、Mailbox、Team Protocol、Autonomous Agent、Worktree | 7 |
| 外部能力 | MCP、Plugin、Resources、Channels、完整 harness 组合 | 8 |

## 推荐阅读路线

如果只想快速建立骨架，先读：

1. [从最小 Agent Loop 开始：Claude Code-like Harness 的骨架](01-从最小-Agent-Loop-开始：Claude-Code-like-Harness-的骨架.md)
2. [工具系统：模型为什么能从“说”变成“做”](02-工具系统：模型为什么能从说变成做.md)
3. [权限系统：Agent 自主行动前必须有边界](03-权限系统：Agent-自主行动前必须有边界.md)
4. [MCP 与完整 Harness：把外部能力接进同一个工具池](08-MCP-与完整-Harness：把外部能力接进同一个工具池.md)

如果想系统理解长任务和团队协作，按完整顺序读 1-8 篇。

如果你是团队技术负责人，建议重点读：

| 关注点 | 推荐文章 |
| --- | --- |
| 安全和权限边界 | 3、4、8 |
| 上下文和成本控制 | 5、8 |
| 长任务不中断 | 6 |
| 多 agent 并行风险 | 7 |
| MCP / Plugin 接入治理 | 8 |

## 系列文章

| 编号 | 文章 | 对应章节 | 配套实验 | 核心问题 |
| --- | --- | --- | --- | --- |
| 1 | [从最小 Agent Loop 开始：Claude Code-like Harness 的骨架](01-从最小-Agent-Loop-开始：Claude-Code-like-Harness-的骨架.md) | s01、s02、s20 | AHE-001、AHE-002 | 一个模型如何变成能持续行动的 agent？ |
| 2 | [工具系统：模型为什么能从“说”变成“做”](02-工具系统：模型为什么能从说变成做.md) | s02、s19、s20 | AHE-002、AHE-003 | 为什么新增工具不应该改主循环？ |
| 3 | [权限系统：Agent 自主行动前必须有边界](03-权限系统：Agent-自主行动前必须有边界.md) | s03、s15、s18、s19 | AHE-004、AHE-016 | Agent 的行动边界在哪里生效？ |
| 4 | [Hooks：把确定性控制挂到 Agent Loop 外面](04-Hooks：把确定性控制挂到-Agent-Loop-外面.md) | s04、s20 | AHE-005、AHE-016 | 哪些控制不该写进提示词？ |
| 5 | [上下文工程：Todo、Skill、Compact、Memory 怎么协同](05-上下文工程：Todo、Skill、Compact、Memory-怎么协同.md) | s05、s07、s08、s09、s10 | AHE-006、AHE-007、AHE-008 | 多个 context 机制分别解决什么问题？ |
| 6 | [失败恢复与长任务：Agent 如何不中途崩掉](06-失败恢复与长任务：Agent-如何不中途崩掉.md) | s11、s12、s13、s14 | AHE-006、AHE-008、AHE-009、AHE-010、AHE-011 | 长任务为什么不能只靠一次会话硬撑？ |
| 7 | [多 Agent 协作：Subagent、Mailbox、Team Protocol、Worktree](07-多-Agent-协作：Subagent、Mailbox、Team-Protocol、Worktree.md) | s06、s12、s15、s16、s17、s18 | AHE-012、AHE-013、AHE-014 | 多 agent 协作靠什么避免互相污染？ |
| 8 | [MCP 与完整 Harness：把外部能力接进同一个工具池](08-MCP-与完整-Harness：把外部能力接进同一个工具池.md) | s19、s20 | AHE-015、AHE-016 | 外部能力如何接进同一个 harness？ |

## 实验路线

实验不要直接在真实项目根目录运行。建议新建临时目录或临时 clone：

```text
/tmp/agent-harness-lab/
```

建议按这个顺序实验：

| 阶段 | 目标 | 重点观察 |
| --- | --- | --- |
| s01-s02 | 跑通最小 loop 和工具调用 | `tool_use` 如何变成 `tool_result` |
| s03-s04 | 加权限和 hooks | 危险动作是否被确定性拦截 |
| s05-s10 | 做上下文实验 | todo、skill、compact、memory 的生命周期差异 |
| s11-s14 | 模拟长任务 | retry、background、cron 是否回到同一 loop |
| s15-s18 | 模拟团队协作 | mailbox、protocol、owner、worktree 是否防止污染 |
| s19-s20 | 接入 MCP 并跑完整 harness | MCP 工具是否进入统一工具池 |

涉及 shell、文件写入、worktree、部署类 MCP 工具时，先用只读或 mock server 验证。

## 和官方能力系列的关系

`tracks/claude-code-public-mechanisms/` 从 Claude Code 官方公开能力出发，回答“Claude Code 公开支持什么、使用边界在哪里”。

本系列从社区教学代码出发，回答“如果要构建一个 Claude Code-like Agent Harness，工程结构大概怎么分层”。

两条线互补：

| Track | 主要材料 | 适合回答 |
| --- | --- | --- |
| Claude Code 官方公开能力系列 | 官方文档和公开能力 | 当前 Claude Code 怎么用、哪些能力处于什么状态 |
| Agent Harness 工程学习系列 | 社区教学代码 + 官方边界 + 作者推导 | Agent runtime 为什么要这样设计、各机制如何组合 |

## 后续扩展方向

这组文章完成后，适合继续补三类材料：

| 方向 | 产物位置 | 内容 |
| --- | --- | --- |
| 术语表 | `references/` | Agent Loop、Tool Runtime、Context Plane、Control Plane、MCP 等术语 |
| 实验手册 | `labs/agent-harness-engineering/` | 每章可复现实验、输入、预期观察、验证标准 |
| 案例研究 | 新 track 或本 track 补充文章 | MCP server 设计、多 agent 项目协作、权限策略模板 |

## 写作纪律

- 每篇开头必须说明：本文基于社区教学项目，不代表 Anthropic Claude Code 官方内部实现。
- 不流水账式逐章摘抄 README；每篇围绕一个核心工程问题展开。
- 每篇至少包含“机制解释”和“最小代码理解”。
- 代码只引用最小必要片段，优先解释结构、控制流和设计意图。
- 每篇末尾提供“动手实验”或“自测问题”。
- 涉及官方 Claude Code 能力时，必须写明官方文档核查日期和参考链接。
