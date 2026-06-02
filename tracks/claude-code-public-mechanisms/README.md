# Claude Code 官方公开能力机制系列

> 当前状态：8 篇初稿完成。  
> 资料边界：以 Claude Code 官方文档、Anthropic 官方仓库和可复现实践为主。  
> 注意：正文中的官方事实以各文章标注的文档核查日期为准，复用前需要重新核查。

## 系列定位

这个 track 用来理解 Claude Code 的公开能力和使用边界。它不讨论 Claude Code 的非公开内部实现，而是从官方文档和可观察实践出发，建立一张面向使用者、团队负责人和平台工程师的机制地图。

## 文章列表

| 编号 | 文章 | 核心问题 |
| --- | --- | --- |
| 1 | [Claude Code 不是聊天工具，而是 Agentic Harness](01-Claude-Code-不是聊天工具，而是-Agentic-Harness.md) | Claude Code 的本质是什么？ |
| 2 | [Claude Code 的记忆不是“记住”，而是一次 Context 装载工程](02-Claude-Code-的记忆不是“记住”，而是一次-Context-装载工程.md) | 新 session 如何重新获得项目上下文？ |
| 3 | [工具系统：Claude Code 为什么能“自己动手”](03-Claude-Code-工具系统为什么能“自己动手”.md) | 工具如何把模型从回答变成执行？ |
| 4 | [权限、安全与 Prompt Injection：Agent 的边界在哪里](04-Claude-Code-权限、安全与-Prompt-Injection：Agent-的边界在哪里.md) | 能行动的 agent 应该被什么边界约束？ |
| 5 | [Hooks 与 MCP：一个管控制，一个连接外部世界](05-Claude-Code-Hooks-与-MCP：一个管控制，一个连接外部世界.md) | Hooks 和 MCP 为什么不能混为一谈？ |
| 6 | [多 Agent 架构：Subagents、Agent Teams 与 Worktrees](06-Claude-Code-多-Agent-架构：Subagents、Agent-Teams-与-Worktrees.md) | 多 Agent 如何隔离 context、任务和工作区？ |
| 7 | [从个人配置到团队标准化：Skills、Plugins 与 Marketplaces](07-Claude-Code-从个人配置到团队标准化：Skills、Plugins-与-Marketplaces.md) | 个人经验如何变成可复用、可分发、可治理的团队能力？ |
| 8 | [把 Claude Code 接入工程系统：GitHub Actions、Headless 与 Agent SDK](08-Claude-Code-把-Claude-Code-接入工程系统：GitHub-Actions、Headless-与-Agent-SDK.md) | Claude Code 如何进入 PR、CI、脚本和自定义应用？ |

## 相关材料

- 实践题库：[labs/claude-code/00-Claude-Code-实践练习题库.md](../../labs/claude-code/00-Claude-Code-实践练习题库.md)
- 原写作规划：[meta/claude-code-public-mechanisms/00-Claude-Code-八篇文章写作规划.md](../../meta/claude-code-public-mechanisms/00-Claude-Code-八篇文章写作规划.md)
- 原学习路线：[meta/claude-code-public-mechanisms/00-Claude-Code-系统学习路线.md](../../meta/claude-code-public-mechanisms/00-Claude-Code-系统学习路线.md)

