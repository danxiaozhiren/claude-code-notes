# Agent Harness 工程学习系列

> 社区项目核查日期：2026-06-08
> 写作仓库：`danxiaozhiren/claude-code-notes`
> 学习对象：`shareAI-lab/learn-claude-code`
> 系列定位：通过 `learn-claude-code` 社区教学代码理解 Claude Code-like Agent Harness 的通用工程机制。

## 边界说明

`shareAI-lab/learn-claude-code` 是社区教学项目，不是 Anthropic Claude Code 官方源码，也不代表 Claude Code 官方内部实现。

本系列写作时统一区分四类信息：

| 类型 | 含义 | 使用方式 |
| --- | --- | --- |
| 官方事实 | 来自 Anthropic / Claude Code 官方文档或官方仓库 | 只用于确认 Claude Code 的公开能力和概念边界 |
| 社区教学实现 | 来自 `shareAI-lab/learn-claude-code` 的 README 与 `code.py` | 作为可运行教学样本分析 |
| 作者推导 | 基于官方事实、教学代码和通用工程经验形成的解释 | 明确标注为推导，不写成官方结论 |
| 动手观察 | 读者可在本地复现实验中看到的现象 | 用输入、预期观察、验证标准组织 |

## 系列文章

| 编号 | 文章 | 对应章节 | 核心问题 |
| --- | --- | --- | --- |
| 1 | [从最小 Agent Loop 开始：Claude Code-like Harness 的骨架](01-从最小-Agent-Loop-开始：Claude-Code-like-Harness-的骨架.md) | s01、s02、s20 | 一个模型如何变成能持续行动的 agent？ |
| 2 | [工具系统：模型为什么能从“说”变成“做”](02-工具系统：模型为什么能从说变成做.md) | s02、s19、s20 | 为什么新增工具不应该改主循环？ |
| 3 | [权限系统：Agent 自主行动前必须有边界](03-权限系统：Agent-自主行动前必须有边界.md) | s03、s15、s18、s19 | Agent 的行动边界在哪里生效？ |
| 4 | Hooks：把确定性控制挂到 Agent Loop 外面 | s04、s20 | 哪些控制不该写进提示词？ |
| 5 | 上下文工程：Todo、Skill、Compact、Memory 怎么协同 | s05、s07、s08、s09、s10 | 多个 context 机制分别解决什么问题？ |
| 6 | 失败恢复与长任务：Agent 如何不中途崩掉 | s11、s12、s13、s14 | 长任务为什么不能只靠一次会话硬撑？ |
| 7 | 多 Agent 协作：Subagent、Mailbox、Team Protocol、Worktree | s06、s12、s15、s16、s17、s18 | 多 agent 协作靠什么避免互相污染？ |
| 8 | MCP 与完整 Harness：把外部能力接进同一个工具池 | s19、s20 | 外部能力如何接进同一个 harness？ |

## 写作纪律

- 每篇开头必须说明：本文基于社区教学项目，不代表 Anthropic Claude Code 官方内部实现。
- 不流水账式逐章摘抄 README；每篇围绕一个核心工程问题展开。
- 每篇至少包含“机制解释”和“最小代码理解”。
- 代码只引用最小必要片段，优先解释结构、控制流和设计意图。
- 每篇末尾提供“动手实验”或“自测问题”。
- 涉及官方 Claude Code 能力时，必须写明官方文档核查日期和参考链接。
