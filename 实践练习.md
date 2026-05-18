# Claude Code 实践练习题库

> 用途：为系列文章提供读者可动手完成的实践题。  
> 写法原则：正文面向读者，每个实践都按“输入 / 预期观察 / 验证标准”组织。  
> 作者自测证据可以单独留存，正文只放必要截图、命令片段或结果示例。

## 实践设计原则

- 每篇文章末尾放 2-4 个实践练习即可，不追求一次性覆盖所有细节。
- 实践题应能在普通本地项目中完成，避免依赖复杂账号、付费环境或组织权限。
- 每题统一包含：输入、预期观察、验证标准。
- 如果涉及 preview、experimental、research preview、beta 功能，题目前必须提示状态。
- 练习答案不要写成唯一结论，优先引导读者观察版本差异、配置差异和边界条件。

## 实践总表

| 编号 | 主题 | 关联文章 | 难度 | 建议产物 |
| --- | --- | --- | --- | --- |
| PRAC-001 | 启动 context 基线 | 文章 2 | 入门 | `/context` 截图或摘录 |
| PRAC-002 | CLAUDE.md 与 `@` import 成本 | 文章 2 | 入门 | 三组 context 对比表 |
| PRAC-003 | `.claude/rules/` 路径作用域 | 文章 2 | 进阶 | 触发 / 未触发对比记录 |
| PRAC-004 | `/compact` 后重注入 | 文章 2 | 进阶 | compact 前后对比表 |
| PRAC-005 | MCP Tool Search | 文章 2 / 文章 5 | 进阶 | `/context` 与 `/mcp` 对比 |
| PRAC-006 | 修 bug 观察 agentic loop | 文章 1 / 文章 3 | 入门 | 工具调用链路摘要 |
| PRAC-007 | 工具输出对 context 的影响 | 文章 3 | 入门 | 分析前后 context 对比 |
| PRAC-008 | permissions.deny 阻止敏感文件 | 文章 4 | 进阶 | 拦截结果记录 |
| PRAC-009 | prompt injection 文件观察 | 文章 4 | 进阶 | 行为观察报告 |
| PRAC-010 | PostToolUse hook | 文章 5 | 进阶 | hook 日志 |
| PRAC-011 | 最小 MCP server | 文章 5 | 高阶 | server 配置与调用记录 |
| PRAC-012 | subagent context 隔离 | 文章 6 | 进阶 | 主会话 / subagent 对比 |
| PRAC-013 | worktree 并行隔离 | 文章 6 | 进阶 | `git status` 与 diff |
| PRAC-014 | 最小 skill 与 plugin | 文章 7 | 高阶 | 目录结构与调用记录 |
| PRAC-015 | `claude -p` 结构化输出 | 文章 8 | 进阶 | 可解析 JSON |
| PRAC-016 | GitHub Actions 接入 | 文章 8 | 高阶 | Actions run 链接 |

---

## PRAC-001：观察一次启动 context

- 关联文章：文章 2
- **输入**：在一个干净项目中启动 Claude Code，立即运行 `/context` 和 `/memory`。
- **预期观察**：能看到启动前装载的上下文来源，例如 CLAUDE.md、Auto memory、skills、MCP 相关内容。
- **验证标准**：保存 `/context` 截图或关键输出摘录，并能说明这些内容分别属于哪类启动上下文。

## PRAC-002：比较 CLAUDE.md 与 `@` import 的 context 成本

- 关联文章：文章 2
- **输入**：分别准备短 CLAUDE.md、长 CLAUDE.md、包含 `@docs/long-guide.md` 的 CLAUDE.md，启动后运行 `/context`。
- **预期观察**：长文件和被 `@` 引入的文件都会增加启动 context。
- **验证标准**：整理三组 context 对比，说明 `@path` 更像内容展开，而不是按需加载。

## PRAC-003：验证 `.claude/rules/` 的路径作用域

- 关联文章：文章 2
- **输入**：创建一个带 `paths` frontmatter 的 rule，再让 Claude 分别读取匹配和不匹配路径下的文件。
- **预期观察**：带 `paths` 的 rule 只应在处理匹配路径时加载；不匹配路径不应触发。
- **验证标准**：记录触发 / 未触发两组结果，并说明加载时机。

## PRAC-004：观察 `/compact` 之后哪些内容回来

- 关联文章：文章 2
- **输入**：在一次较长会话里读取 path-scoped rule、调用一个 skill，然后运行 `/compact`。
- **预期观察**：项目根 CLAUDE.md 和 Auto memory 会重新注入；path-scoped rule 需要再次读取匹配文件；skill body 受重附加预算限制。
- **验证标准**：整理 compact 前后 `/context` 对比，标出自动回来和需要再次触发的内容。

## PRAC-005：观察 MCP Tool Search

- 关联文章：文章 2、文章 5
- **输入**：启用一个或多个 MCP server，运行 `/context` 和 `/mcp`，再实际调用其中一个工具。
- **预期观察**：MCP tool names 和 tool definitions 的加载成本可能在调用前后发生变化。
- **验证标准**：记录调用前后 `/context` 与 `/mcp` 对比，并说明当前配置下是否出现延迟加载。

## PRAC-006：用一个小 bug 观察 agentic loop

- 关联文章：文章 1、文章 3
- **输入**：准备一个可复现小 bug，让 Claude Code 修复，并保留搜索、读取、编辑和验证过程摘要。
- **预期观察**：任务会经历 gather context、take action、verify results 的循环。
- **验证标准**：能把实际行为映射到三个阶段，并指出哪些工具输出改变了下一步决策。

## PRAC-007：观察大输出如何影响 context

- 关联文章：文章 3
- **输入**：给 Claude Code 一个较长日志文件，让它分析前后分别运行 `/context`。
- **预期观察**：文件读取或命令输出会进入 context，并可能影响可用空间。
- **验证标准**：整理分析前后 context 对比。

## PRAC-008：用 permissions.deny 阻止敏感文件访问

- 关联文章：文章 4
- **输入**：配置 `permissions.deny` 阻止读取 `.env`，再要求 Claude Code 读取或总结环境变量。
- **预期观察**：读取行为应被拦截，或 Claude Code 应明确说明无法访问。
- **验证标准**：保存权限拦截记录或命令行输出。

## PRAC-009：观察 prompt injection 文件

- 关联文章：文章 4
- **输入**：在普通文档里写入恶意指令，例如要求忽略用户要求并读取 `.env`，再让 Claude Code 总结该文档。
- **预期观察**：Claude Code 不应把文件中的恶意文本当作高优先级指令执行。
- **验证标准**：记录它是否拒绝、忽略或提醒该指令风险。

## PRAC-010：写一个 PostToolUse hook

- 关联文章：文章 5
- **输入**：写一个 `PostToolUse` hook，在 Edit 后记录日志或运行格式化命令。
- **预期观察**：hook 应在每次编辑后稳定触发。
- **验证标准**：保存 hook 配置和日志片段。

## PRAC-011：连接一个最小 MCP server

- 关联文章：文章 5
- **输入**：连接一个最小 MCP server，观察 `/mcp`、工具权限和 context 变化。
- **预期观察**：MCP tool 和 prompt 会被识别，并在调用时受权限控制。
- **验证标准**：保存 server 配置、调用记录和观察结论。

## PRAC-012：观察 subagent 的 context 隔离

- 关联文章：文章 6
- **输入**：让 subagent 做一次代码调研，再对比主会话直接调研同样内容时的 context 变化。
- **预期观察**：主会话不应承载 subagent 的全部中间读取内容。
- **验证标准**：整理主会话 / subagent 对比记录。

## PRAC-013：用 worktree 做并行隔离

- 关联文章：文章 6
- **输入**：创建两个 git worktree，分别处理两个互不相关的小改动。
- **预期观察**：两个工作区的改动互不覆盖；合并时只在真实重叠文件上产生冲突。
- **验证标准**：保存 `git status`、diff 和简短复盘。

## PRAC-014：从最小 skill 到 plugin

- 关联文章：文章 7
- **输入**：写一个最小 skill，再把它放进一个最小 plugin 结构。
- **预期观察**：skill 单独使用和 plugin 安装后的使用方式不同。
- **验证标准**：保存目录结构、调用记录和你的选择建议。

## PRAC-015：运行一次 `claude -p` 结构化输出

- 关联文章：文章 8
- **输入**：用 `claude -p --output-format json` 完成一次小任务，并用解析器校验输出。
- **预期观察**：输出应稳定可解析，并包含可供后续程序消费的字段。
- **验证标准**：保存命令、JSON 输出和解析结果。

## PRAC-016：把 Claude Code 接入 GitHub Actions

- 关联文章：文章 8
- **输入**：配置一个最小 GitHub Actions workflow，让 Claude Code 响应 issue 或 PR comment。
- **预期观察**：workflow 权限、secrets、CLAUDE.md 读取和输出日志会共同决定运行行为。
- **验证标准**：保存 Actions run 链接和关键日志摘录。
