# Claude Code 系统学习路线

> 基于 Claude Code 官方文档结构整理。  
> 校准日期：2026-05-17。

## 学习主线

Claude Code 不只是一个命令行聊天工具，而是一个围绕 Claude 模型构建的 **agentic harness / agent runtime**。

系统学习时，不应只按命令或功能点记忆，而应围绕这条主线展开：

```text
Core concepts
  -> Instructions / Memory / Context
  -> Tools / Permissions / Security
  -> Hooks / MCP / Automation
  -> Subagents / Agent Teams / Worktrees
  -> Skills / Plugins / Marketplaces
  -> CI/CD
  -> Agent SDK
```

---

## 第一层：核心概念

### 1. How Claude Code works

文档：[How Claude Code works](https://code.claude.com/docs/en/how-claude-code-works)

先建立总模型：Claude Code 的核心是 agentic loop，而不是一次性问答。

官方将它描述为三个阶段循环推进：

- gather context
- take action
- verify results

重点理解：Claude Code 是围绕 Claude 的 agentic harness，负责提供工具、context 管理和执行环境。

### 2. Extend Claude Code

文档：[Extend Claude Code](https://code.claude.com/docs/en/features-overview)

这页是扩展系统地图，应在学习具体功能前先读。

它定义了这些能力的边界：

- CLAUDE.md
- Skills
- MCP
- Subagents
- Agent Teams
- Hooks
- Plugins

核心问题不是“每个功能怎么用”，而是“什么时候该用哪个功能”。

### 3. Explore the context window

文档：[Explore the context window](https://code.claude.com/docs/en/context-window)

理解一次 session 中 context 如何被填充、消耗、压缩和重注入。

这是理解记忆系统、skills、subagents、MCP 成本和 compact 行为的基础。

---

## 第二层：记忆与指令系统

### 4. Store instructions and memories

文档：[How Claude remembers your project](https://code.claude.com/docs/en/memory)

重点学习：

- CLAUDE.md 的层级与加载规则
- Auto memory 的写入和加载方式
- `.claude/rules/` 的路径作用域规则
- compact 之后哪些内容会保留，哪些内容会丢失

这一层的核心结论：记忆不是“模型记住了”，而是一次 context 装载工程。

### 5. Skills

文档：[Extend Claude with skills](https://code.claude.com/docs/en/skills)

Skills 不是 CLAUDE.md 的附属品，而是按需加载的知识和工作流单元。

重点学习两种用法：

- reference skill：提供知识、规范、参考资料
- action skill：封装可调用工作流，例如 `/deploy`、`/review`

还要理解 `disable-model-invocation`：它控制 skill 是由模型自动触发，还是只能由用户显式调用。

---

## 第三层：工具与权限

### 6. Tools available to Claude

文档：[Tools reference](https://code.claude.com/docs/en/tools-reference)

重点不是背工具清单，而是理解工具如何参与 agent loop。

Claude Code 的内置工具大致包括：

- file operations
- search
- execution
- web
- code intelligence

每次工具调用的结果都会反馈给模型，成为下一步判断的依据。

### 7. Permission modes and permissions

文档：

- [Permission modes](https://code.claude.com/docs/en/permission-modes)
- [Configure permissions](https://code.claude.com/docs/en/permissions)
- [Security](https://code.claude.com/docs/en/security)

重点学习：

- Default
- Auto-accept edits
- Plan mode
- Auto mode
- allow / ask / deny 规则
- sandbox 与 permissions 的区别
- prompt injection 防护
- sensitive files deny rules
- managed settings

注意：Auto mode 是 research preview，应作为实验性能力看待。

---

## 第四层：扩展与自动化

### 8. Hooks

文档：

- [Automate workflows with hooks](https://code.claude.com/docs/en/hooks-guide)
- [Hooks reference](https://code.claude.com/docs/en/hooks)

Hooks 是确定性控制层，运行在模型判断之外。

适合处理“不应该交给模型自由发挥”的事情：

- 编辑后自动格式化
- 阻止危险 Bash 命令
- 加载额外上下文
- 记录审计日志
- 在 compact 前后执行动作

核心判断：如果一件事必须每次稳定发生，就应该用 hook，而不是只写进提示词。

### 9. MCP

文档：[Connect Claude Code to tools via MCP](https://code.claude.com/docs/en/mcp)

MCP 是外部能力连接层，让 Claude Code 可以连接 GitHub、Slack、数据库、浏览器、内部工具等。

重点学习：

- MCP server 如何暴露 tools
- MCP prompts 如何成为 slash commands
- MCP 权限如何管理
- 组织级 managed MCP 配置
- MCP 与 Skills 的区别：MCP 提供能力，Skills 提供使用能力的知识和流程

### 10. Automation

文档：

- [Channels](https://code.claude.com/docs/en/channels)
- [Scheduled tasks](https://code.claude.com/docs/en/scheduled-tasks)
- [Goals](https://code.claude.com/docs/en/goal)
- [Run Claude Code programmatically](https://code.claude.com/docs/en/headless)

这是无人值守和自动化场景的基础。

重点学习：

- Channels：外部事件推送到运行中的 session
- Scheduled tasks：定时运行 prompt
- Goals：让 Claude 持续工作直到满足目标
- Headless：用 `claude -p` 进行非交互式运行

---

## 第五层：多 Agent 系统

### 11. Subagents

文档：[Create custom subagents](https://code.claude.com/docs/en/sub-agents)

Subagent 是当前 session 内的隔离 worker。

核心价值：

- 独立 context window
- 大量搜索和读取不污染主会话
- 最终只把 summary 回传主 context

适合研究、审查、验证、并行调查等任务。

### 12. Agent Teams

文档：[Agent Teams](https://code.claude.com/docs/en/agent-teams)

Agent Teams 是多个独立 Claude Code sessions 的协作系统。

和 subagent 的区别：

- Subagent 是主 agent 管理的子任务
- Agent Team 是多个 session 之间的协作
- 支持 shared task list 和 peer-to-peer messaging

注意：Agent Teams 是实验性功能，默认关闭。

### 13. Worktrees

文档：[Run parallel sessions with worktrees](https://code.claude.com/docs/en/worktrees)

Worktree 是并行开发的文件系统隔离手段。

重点学习：

- 每个 Claude session 在独立 git worktree 中工作
- 不同任务的文件改动互不冲突
- worktree 可以和 subagents、parallel sessions 组合使用

---

## 第六层：打包与分发

### 14. Plugins and marketplaces

文档：

- [Create plugins](https://code.claude.com/docs/en/plugins)
- [Plugin marketplaces](https://code.claude.com/docs/en/plugin-marketplaces)

Plugin 是团队工程化 Claude Code 用法的分发单元。

一个 plugin 可以包含：

- skills
- agents
- hooks
- MCP servers
- settings
- commands
- bin executables

Marketplaces 则用于跨团队、跨项目分发插件。

重点理解：当同一套 Claude Code 配置需要在多个仓库复用时，应考虑 plugin 化，而不是复制粘贴 `.claude/` 目录。

---

## 第七层：CI/CD 与 Agent SDK

### 15. GitHub Actions and code review

文档：

- [Claude Code GitHub Actions](https://code.claude.com/docs/en/github-actions)
- [Code Review](https://code.claude.com/docs/en/code-review)

重点学习 Claude Code 如何进入 PR / Issue / CI 流程：

- `@claude` 响应 issue 或 PR comment
- 自动实现简单需求
- 自动生成 PR
- 自动 review
- 安全审查
- 工作流权限和 secrets 管理

GitHub 参考：

- [anthropics/claude-code-action](https://github.com/anthropics/claude-code-action)
- [anthropics/claude-code-security-review](https://github.com/anthropics/claude-code-security-review)

### 16. Agent SDK

文档：

- [Agent SDK overview](https://code.claude.com/docs/en/agent-sdk/overview)
- [How the agent loop works](https://code.claude.com/docs/en/agent-sdk/agent-loop)

官方定位：把 Claude Code 作为库，用来构建生产级 AI agent。

重点学习：

- agent loop 控制
- sessions
- streaming input / output
- structured output
- custom tools
- MCP
- hooks
- permissions
- subagents
- cost tracking
- observability

GitHub 参考：

- [anthropics/claude-agent-sdk-python](https://github.com/anthropics/claude-agent-sdk-python)
- [anthropics/skills](https://github.com/anthropics/skills)

---

## 第一个实测问题

### MCP 工具定义是否仍然启动时全量加载？

旧认知：MCP 工具名称和描述在 session 启动时全量加载，可能成为启动 context 的主要隐性开销。

官方最新文档中出现了一个需要校验的新说法：MCP tool definitions 默认 deferred，通过 tool search 按需加载。

因此不能直接沿用旧结论，应实测验证：

1. 在只启用少量 MCP server 的情况下运行 `/context`
2. 在启用大量 MCP server 的情况下运行 `/context`
3. 使用 `/mcp` 查看 per-server cost
4. 实际调用某个 MCP tool 后，再次观察 `/context`
5. 对比 tool definitions 是否在使用前后发生明显变化

验证目标：

- MCP server 名称是否仍在启动时进入 context
- MCP tool description 是否延迟加载
- tool search 是否改变了旧版“全量工具描述占 context”的结论
- 不同版本 Claude Code 的行为是否存在差异

---

## 推荐阅读顺序

```text
1. how-claude-code-works
2. features-overview
3. context-window
4. memory
5. skills
6. tools-reference
7. permission-modes / permissions / security
8. hooks-guide / hooks
9. mcp
10. channels / scheduled-tasks / goal / headless
11. sub-agents
12. agent-teams
13. worktrees
14. plugins / plugin-marketplaces
15. github-actions / code-review
16. agent-sdk overview / agent-loop
```

---

## 一句话总结

学习 Claude Code 的正确路径，不是从“命令怎么用”开始，而是从它作为 **agentic harness** 的结构开始：

```text
模型负责推理，
工具负责行动，
context 负责工作记忆，
permissions 负责边界，
hooks 负责确定性控制，
MCP 负责外部能力，
subagents / teams / worktrees 负责并行和隔离，
plugins / SDK 负责工程化和产品化。
```
