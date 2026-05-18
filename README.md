# Claude Code Notes

> 当前定位：围绕 Claude Code 的系统学习、文章写作与读者实践资料库。  
> 作者：数据室 伍涛。  
> 当前仓库快照：2026-05-18。  
> 既有文章的官方文档核查日期以正文标注为准，复用或发布前需要重新核查。

## 项目目标

这个仓库用于把 Claude Code 的系统学习沉淀成一组可发布、可复核、可迭代的中文文章。

写作初衷：在斌总和葛处的鼓励与倡导下，围绕 Claude Code 做一次面向底层原理的系统学习，把学习过程中的理解、积累和实践经验沉淀下来，并以文章形式分享给更多同事和读者。

核心主线是：

```text
Claude Code 不是普通聊天工具，
而是围绕 Claude 模型构建的 agentic harness / agent runtime。
```

写作时优先把每个结论落到四类来源之一：

- 官方事实：来自 `code.claude.com/docs` 或 Anthropic 官方仓库。
- 社区实践：来自可信社区仓库或可复现案例。
- 作者推导：基于官方事实和工程经验形成的解释。
- 动手验证：设计成读者可以复现的实践练习；作者自测证据可内部留存，不必在正文里暴露内部记录状态。

## 当前文件

- `Claude Code 系统学习路线.md`：总纲，按学习层次拆解 Claude Code 的核心概念、context、工具、安全、扩展、多 Agent、CI/CD 和 Agent SDK。
- `Claude Code 八篇文章写作规划.md`：系列写作蓝图，包含 8 篇文章的目标读者、核心问题、参考资料和实践练习。
- `Claude Code 不是聊天工具，而是 Agentic Harness.md`：系列第 1 篇初稿，建立 Claude Code 作为 agentic harness / agent runtime 的总模型。
- `Claude Code 的记忆不是“记住”，而是一次 Context 装载工程.md`：系列第 2 篇初稿，聚焦 memory、instructions、rules、skills、MCP 与 context 装载机制。
- `实践练习.md`：读者实践题库，用于把 `/context`、`/mcp`、`/memory`、`/hooks`、`claude -p`、GitHub Actions 等验证项改写成可操作练习。

## 写作进度

| 编号 | 文章 | 状态 | 下一步 |
| --- | --- | --- | --- |
| 1 | Claude Code 不是聊天工具，而是 Agentic Harness | 初稿完成 | 通读校准官方事实，准备发布前润色 |
| 2 | Claude Code 的记忆不是“记住”，而是 Context 装载工程 | 初稿完成 | 通读校准来源标识和实践练习 |
| 3 | 工具系统：Claude Code 为什么能“自己动手” | 规划完成 | 撰写初稿，区分静态 context 与工具输出动态填充 |
| 4 | 权限、安全与 Prompt Injection：Agent 的边界在哪里 | 规划完成 | 设计 permissions、sensitive files、prompt injection 实践练习 |
| 5 | Hooks 与 MCP：一个管控制，一个连接外部世界 | 规划完成 | 设计最小 hook 和最小 MCP server 实践练习 |
| 6 | 多 Agent 架构：Subagents、Agent Teams 与 Worktrees | 规划完成 | 补决策树和 worktree / subagent 对比实践 |
| 7 | 从个人配置到团队标准化：Skills、Plugins 与 Marketplaces | 规划完成 | 设计最小 skill、plugin 和 marketplace 实践 |
| 8 | 把 Claude Code 接入工程系统：GitHub Actions、Headless 与 Agent SDK | 规划完成 | 设计 `claude -p`、structured output 和 GitHub Actions 实践 |

## 推荐推进顺序

1. 收口第 1、2 篇：它们已经成文，优先做发布前事实核查、术语统一和细节润色。
2. 撰写第 3 篇：承接第 1、2 篇，解释工具输出如何驱动 agent loop。
3. 补一篇轻量导读：为读者说明推荐阅读顺序和实践方式。
4. 补安全与扩展专题：权限、hooks、MCP、多 Agent、plugins、CI/CD 按规划逐篇推进。

## 写作纪律

- 每篇文章头部必须标注文档核查日期。
- experimental、preview、research preview、beta 功能首次出现时必须标注状态。
- 不把社区经验包装成官方结论。
- 不把旧版本行为直接当作当前事实。
- 不把模型行为经验描述成强保证。
- 正文面向读者时优先写“实践练习”“动手验证”，不要暴露作者内部待办状态。

## 发布前检查

- 官方链接仍然有效，且关键结论已按最新文档校准。
- 每个实践练习都按 `输入 / 预期观察 / 验证标准` 组织。
- 易错点里明确区分官方事实、社区实践、作者推导和实践观察。
- 文章末尾有“实践练习”或“动手验证”，而不是作者式的“实测结果摘要”。
- 没有未标注的 preview / experimental 功能。
