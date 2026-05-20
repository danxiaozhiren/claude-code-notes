# 把 Claude Code 接入工程系统：GitHub Actions、Headless 与 Agent SDK

> 文档核查日期：2026-05-20  
> 作者：wt  
> 写作初衷：在领导的鼓励与倡导下，围绕 Claude Code 做一次面向底层原理的系统学习，把学习过程中的理解、积累和实践经验沉淀下来，并以文章形式分享给更多同事和读者。  
> 适合读者：团队负责人、平台开发者、DevOps / 平台工程师  
> 本文定位：Claude Code 系列第 8 篇，承接前七篇的 context、工具、权限、Hooks / MCP、多 Agent 和团队标准化，解释 Claude Code 如何进入 PR、CI、脚本和自定义 agent 产品。  
> 实践建议：本文末尾提供动手练习，建议读者先用 `claude -p` 做一次可解析输出，再配置最小 GitHub Actions，最后用 Agent SDK 写一个只读 agent。

---

## 先把边界说清楚

前七篇主要讨论 Claude Code 在“开发者本地会话”里如何工作。第 8 篇换一个视角：

> 当 Claude Code 进入 CI、PR、脚本和应用系统，它就不再只是一个你坐在旁边看着的工具，而会变成工程系统的一部分。

这篇讲三条路径：

```text
GitHub Actions：把 Claude Code 接到 PR / Issue / CI 事件里。
Headless：用 claude -p 把 Claude Code 当成非交互式命令行程序。
Agent SDK：把 Claude Code 的 agent loop 作为库嵌入自己的应用。
```

不要把它们混成一个概念：

```text
GitHub Actions 解决“仓库事件如何触发 Claude”。
Headless 解决“脚本如何调用 Claude 并拿到输出”。
Agent SDK 解决“应用如何直接控制 agent loop、工具、权限、会话和观测”。
```

本文关注：

- `@claude` 触发 PR / Issue 工作流
- Code Review 和 GitHub Actions 的区别
- GitHub Actions secrets、workflow permissions、超时和成本控制
- `claude -p` 非交互式运行
- `--bare` 模式和可复现性
- `--output-format json`、`stream-json`、`--json-schema`
- Agent SDK 的 `query()`、sessions、custom tools、hooks、subagents
- structured output、cost tracking 与 OpenTelemetry observability

不展开的内容：

- permissions 的完整规则：见第 4 篇
- hooks 和 MCP 的底层机制：见第 5 篇
- subagents 的架构判断：见第 6 篇
- skills / plugins / marketplaces 的团队分发：见第 7 篇

本文只在容易混淆的地方标注来源类型：

- 【官方事实】：来自 Claude Code 官方文档或 Anthropic 官方仓库。
- 【官方 GitHub】：来自 Anthropic 官方 GitHub 仓库。
- 【作者推导】：基于官方事实整理出的机制解释和工程判断。
- 【实践练习】：读者可以在自己的环境中复现观察的部分。

---

## 一、核心认知：工程系统要的是可复现、可限权、可观测

本地使用 Claude Code 时，人是最后一道边界：

```text
Claude 想改文件，你看一下再确认。
Claude 跑命令，你看一下再允许。
Claude 输出建议，你判断是否采纳。
```

进入工程系统后，这个假设会变化。

GitHub Actions 可能在半夜被 PR 触发；`claude -p` 可能被脚本调用；Agent SDK 可能服务多个用户请求。于是问题不再只是“Claude 会不会做”，而是：

```text
谁能触发？
能访问什么？
能修改什么？
输出能不能被机器解析？
失败时怎么追踪？
成本和轮次有没有上限？
结果是否留下可审计产物？
```

【作者推导】把 Claude Code 接进工程系统，不是把“本地 agent”简单搬到服务器上，而是把 agent loop 放进一组工程约束：

| 约束 | 本地会话 | 工程系统 |
| --- | --- | --- |
| 触发 | 人手动输入 | PR、Issue、CI、脚本、API 请求 |
| 上下文 | 当前工作区和用户配置 | runner、container、显式 settings、仓库文件 |
| 权限 | 用户逐步批准 | workflow permissions、allowed tools、permission mode |
| 输出 | 人读文本 | JSON、check run、inline comment、trace、artifact |
| 失败处理 | 人继续追问 | exit code、ResultMessage subtype、CI 日志、告警 |
| 成本控制 | 人停下来 | max turns、budget、timeout、concurrency |

所以本文的核心不是“如何让 Claude 自动化更多”，而是：

```text
如何让 Claude 在工程系统里自动化，同时仍然可复现、可限权、可观测。
```

---

## 二、GitHub Actions：把 Claude 放进 PR / Issue / CI

【官方事实】Claude Code GitHub Actions 可以在 GitHub Actions workflow 中运行 Claude Code。官方文档说明，用户可以在 PR 或 Issue 评论中提到 `@claude`，让 Claude 分析代码、实现功能、修 bug、创建 PR 或回复上下文问题。该 Action 基于 Claude Agent SDK 构建。

这意味着 GitHub Actions 是“事件驱动”的集成方式：

```text
GitHub event
  -> workflow runner
  -> claude-code-action
  -> Claude Code agent loop
  -> comment / commit / PR / check output
```

### 2.1 `@claude` 触发的是仓库里的 agent loop

【官方事实】官方 quick setup 推荐在 Claude Code 终端中运行 `/install-github-app`，它会引导安装 GitHub App 和 required secrets。用户需要 repository admin 权限。手动安装时，Claude GitHub App 会请求 Contents、Issues、Pull requests 的读写权限，并需要在 GitHub Secrets 中配置 `ANTHROPIC_API_KEY`。

最小形态可以理解成：

```yaml
name: Claude Code

on:
  issue_comment:
    types: [created]
  pull_request_review_comment:
    types: [created]

permissions:
  contents: write
  issues: write
  pull-requests: write

jobs:
  claude:
    runs-on: ubuntu-latest
    steps:
      - uses: anthropics/claude-code-action@v1
        with:
          anthropic_api_key: ${{ secrets.ANTHROPIC_API_KEY }}
```

这个 workflow 的直觉是：

```text
在评论里写 @claude
  -> GitHub event 触发 workflow
  -> Claude 在 runner 里运行
  -> Claude 根据 issue / PR / repo context 采取行动
```

【作者推导】这里最容易误解的一点是：`@claude` 不是“云端聊天机器人回复一下”。它会把 Claude Code 放进 GitHub Actions runner 的上下文里运行。Runner 上能 checkout 什么代码、action 有什么 token 权限、workflow 暴露了哪些 secrets，都会决定 Claude 实际能做什么。

### 2.2 自动化 workflow：不是所有场景都需要 `@claude`

【官方事实】Claude Code Action v1 提供统一的 `prompt` 输入，也支持通过 `claude_args` 传递 CLI 参数。没有 `@claude` mention 的自动化场景可以直接在 workflow 中写 prompt，例如定时报告、PR 自动检查、发布前检查等。

示意：

```yaml
name: Claude PR Check

on:
  pull_request:
    types: [opened, synchronize]

permissions:
  contents: read
  pull-requests: write

jobs:
  review:
    runs-on: ubuntu-latest
    timeout-minutes: 20
    steps:
      - uses: actions/checkout@v4
      - uses: anthropics/claude-code-action@v1
        with:
          anthropic_api_key: ${{ secrets.ANTHROPIC_API_KEY }}
          prompt: "Review this pull request for correctness bugs and risky changes."
          claude_args: |
            --max-turns 8
            --allowedTools "Read,Grep,Glob"
```

【官方事实】如果 workflow 要调用仓库中的 skill，需要先 `actions/checkout`，再在 `prompt` 中传入 `/skill-name`。如果 skill 来自 plugin，可以通过 `plugin_marketplaces` 和 `plugins` 安装后，用 namespaced skill 调用。

【作者推导】这正好接上第 7 篇：团队可以先把审查标准沉淀成 skill / plugin，再由 GitHub Actions 调用它。这样标准维护在一个地方，而不是散落在每个 workflow 的长 prompt 里。

### 2.3 Secrets 和 permissions 是 CI 中的第一道边界

【官方事实】官方文档强调不要把 API key 写进仓库，应该使用 GitHub Secrets，例如 `ANTHROPIC_API_KEY`。文档也建议限制 action permissions，只给必要权限，并配置 timeout、`--max-turns` 和 GitHub concurrency 来控制成本和 runaway jobs。

【作者推导】在 CI 中，权限至少有三层：

| 层 | 控制什么 |
| --- | --- |
| GitHub App / token permissions | Claude 能不能读写 contents、issues、pull requests |
| workflow `permissions` | 当前 job 的 `GITHUB_TOKEN` 能做什么 |
| Claude Code `claude_args` / settings | Claude 在 agent loop 里能用哪些 tools、跑多少 turns |

不要只看 Claude Code 的 permissions。CI 里的真实权限是这三层叠加出来的。

保守做法：

- 默认从 read-only workflow 开始
- 写 PR comment 比直接 push 更安全
- 对 fork PR 更谨慎，避免暴露 secrets
- 对写文件、推送、部署类 workflow 加 manual approval 或环境保护
- 给 job 设置 `timeout-minutes`
- 给 Claude 设置 `--max-turns`
- 用 concurrency 限制同一 PR 上的重复运行

### 2.4 Code Review：托管审查服务，不等于普通 GitHub Action

【官方事实】Claude Code 的 Code Review 功能处于 research preview，可用于 Team 和 Enterprise 订阅；Zero Data Retention 组织不可用。它会分析 GitHub PR，并把发现的问题作为 inline comments 发到具体代码行。官方文档说明，它使用多个 specialized agents 在完整代码库上下文中分析逻辑错误、安全漏洞、边界问题和回归风险。

> 注意：Code Review 是 research preview。本文只描述截至 2026-05-20 的官方文档状态，不把它当成稳定行为承诺。

【官方事实】Code Review 可以在 PR 创建、每次 push 或手动触发时运行。手动命令包括：

```text
@claude review
@claude review once
```

两者区别：

| 命令 | 含义 |
| --- | --- |
| `@claude review` | 启动一次审查，并订阅后续 push-triggered reviews |
| `@claude review once` | 只启动一次审查，不订阅后续 push |

【官方事实】Code Review 的 check run 是 neutral conclusion，不会自动阻止 merge。若团队想把 findings 作为合并门禁，需要在自己的 CI 中解析 check run 的输出。

【作者推导】可以这样区分：

| 选择 | 适合场景 |
| --- | --- |
| Code Review | 想要托管式 PR 审查、inline comments、少写 workflow |
| GitHub Actions | 想自定义 prompt、工具权限、触发条件、输出格式、CI gate |
| `claude -p` | 想在任意脚本或现有 CI 中插入一次 Claude 调用 |
| Agent SDK | 想把 agent loop 做成自己的服务或产品能力 |

Code Review 更像“托管审查产品”。GitHub Actions 更像“把 Claude Code 作为 CI 组件”。不要把两者混成同一件事。

---

## 三、Headless：用 `claude -p` 把 Claude Code 变成脚本组件

【官方事实】`claude -p` 或 `claude --print` 会让 Claude Code 以非交互模式运行。官方文档说明，`-p` 可配合 CLI options 使用，例如 `--continue`、`--allowedTools`、`--output-format`。

最小调用：

```bash
claude -p "Summarize this project"
```

带工具权限：

```bash
claude -p "Find risky shell commands in this repo" --allowedTools "Read,Grep,Glob"
```

【官方事实】`--allowedTools` 使用 permission rule syntax。像 `Read,Grep,Glob` 这种是工具级别的预批准；如果要限制 Bash 的具体命令，可以写成 scoped rule：

```bash
claude -p "Look at my staged changes and create an appropriate commit" \
  --allowedTools "Bash(git diff *),Bash(git log *),Bash(git status *),Bash(git commit *)"
```

这里尾部的 ` *` 表示前缀匹配，空格很重要：`Bash(git diff *)` 匹配 `git diff ...`，而 `Bash(git diff*)` 可能误匹配 `git diff-index` 这类命令。

【作者推导】CI 场景里尽量用 scoped rule，而不是直接放开 `Bash`。允许 `Bash` 是给 Claude 一把通用钥匙；允许 `Bash(git diff *)` 只是允许它看 diff。

【作者推导】`claude -p` 的本质是把一次 agent session 变成命令行调用：

```text
stdin / prompt
  -> Claude Code agent loop
  -> stdout / JSON / stream-json
  -> shell script or CI step consumes output
```

这比 GitHub Actions 更底层，也更容易嵌进已有系统。

### 3.1 管道输入和输出格式

【官方事实】非交互模式可以读取 stdin，也可以把响应重定向到文件。官方文档说明，截至 Claude Code v2.1.128，piped stdin 上限是 10MB；更大的输入应该写成文件，再在 prompt 中引用文件路径。

示意：

```bash
git diff main | claude -p "Review this diff for risky changes"
```

【官方事实】`--output-format` 支持：

| 格式 | 适合 |
| --- | --- |
| `text` | 人读文本，默认格式 |
| `json` | 脚本解析 result、session id、usage、cost 等元数据 |
| `stream-json` | 实时处理事件流 |

例如：

```bash
claude -p "Summarize this project" --output-format json
```

如果要看到 token 级的流式事件，需要配合 `--verbose` 和 `--include-partial-messages`：

```bash
claude -p "Explain recursion" \
  --output-format stream-json \
  --verbose \
  --include-partial-messages
```

【作者推导】`json` 更适合“结束后解析结果”，`stream-json` 更适合“运行中更新 UI、日志或进度条”。如果只是 CI gate，通常不需要 token 级流式输出；如果是在 Web UI 里展示 agent 过程，就更值得用 `stream-json`。

若要约束输出结构，可以加 `--json-schema`：

```bash
claude -p "Extract changed modules from this repository" \
  --output-format json \
  --json-schema '{"type":"object","properties":{"modules":{"type":"array","items":{"type":"string"}}},"required":["modules"]}'
```

【作者推导】脚本里不要靠正则解析自然语言。能用 JSON schema，就尽量让输出从一开始就是机器可验证的。

### 3.2 `--bare`：让脚本更可复现

【官方事实】`--bare` 会跳过 hooks、skills、plugins、MCP servers、Auto memory 和 CLAUDE.md 的自动发现。没有 `--bare` 时，`claude -p` 会加载和交互式 session 类似的上下文，包括工作目录和 `~/.claude` 中的配置。官方文档还说明，`--bare` 是 scripted 和 SDK calls 的推荐模式，并会在未来版本成为 `-p` 的默认行为。

示意：

```bash
claude --bare -p "Summarize README.md" --allowedTools "Read"
```

【官方事实】bare mode 只让显式传入的 flags 生效。需要上下文时，可以通过参数传入，例如：

【官方事实】在 bare mode 中，Claude 仍可访问 Bash、文件读取和文件编辑工具；bare mode 只是跳过自动发现的上下文来源，并不等于自动关闭工具能力。工具能否免确认运行，仍由 `--allowedTools`、permission mode 和 settings 决定。

| 要加载什么 | 参数 |
| --- | --- |
| system prompt 补充 | `--append-system-prompt` / `--append-system-prompt-file` |
| settings | `--settings <file-or-json>` |
| MCP servers | `--mcp-config <file-or-json>` |
| custom agents | `--agents <json>` |
| plugin | `--plugin-dir <path>` / `--plugin-url <url>` |

【官方事实】bare mode 会跳过 OAuth 和 keychain 读取。Anthropic 认证需要来自 `ANTHROPIC_API_KEY`，或者来自传给 `--settings` 的 JSON 中的 `apiKeyHelper`。Bedrock、Vertex、Foundry 使用各自 provider credentials。

【作者推导】`--bare` 不是“安全模式”的同义词。它主要解决可复现性：不让脚本偷偷吃到某个开发者本机的 hooks、MCP 或 memory。真正的安全仍然要靠 tools、settings、sandbox、CI 权限和 secrets 管理。

### 3.3 Headless 的工程边界

适合用 `claude -p` 的场景：

- 在 CI 里做一次只读分析
- 对构建日志做摘要
- 对 diff 做风险分类
- 生成机器可解析的 JSON 结果
- 把 Claude 作为现有脚本的一步

不适合直接用 `claude -p` 的场景：

- 需要长期 session 管理
- 需要复杂用户交互
- 需要自定义工具 handler
- 需要细粒度 approval callback
- 需要把 agent loop 嵌进产品服务

这些场景更适合 Agent SDK。

---

## 四、Agent SDK：把 Claude Code 的 agent loop 嵌进应用

【官方事实】Agent SDK 可以在 Python 和 TypeScript 中使用。官方文档说明，它提供和 Claude Code 相同的工具、agent loop 和 context management，让应用能构建会读文件、跑命令、搜索、编辑代码的 agents。TypeScript SDK 会把本机 Claude Code binary 作为 optional dependency 捆绑，因此不一定需要单独安装 Claude Code CLI。

最小 Python 形态：

```python
import asyncio
from claude_agent_sdk import query, ClaudeAgentOptions

async def main():
    async for message in query(
        prompt="List important files in this repository",
        options=ClaudeAgentOptions(allowed_tools=["Read", "Glob"]),
    ):
        print(message)

asyncio.run(main())
```

【官方事实】Agent SDK 支持 Anthropic API key，也支持 Amazon Bedrock、Claude Platform on AWS、Google Vertex AI、Microsoft Azure 等 provider authentication。官方文档还说明，第三方开发者除非获得批准，不应把 claude.ai 登录或订阅额度提供给自己的产品使用；应使用 API key 认证方式。

【官方事实】官方文档在 2026-05-20 的页面中提示：从 2026-06-15 起，subscription plans 上的 Agent SDK 和 `claude -p` 使用会计入新的 monthly Agent SDK credit，和 interactive usage limits 分开。本文不展开计费细则，只提醒读者复用或发布前重新核查。

### 4.1 Agent loop：SDK 暴露的是循环，不只是一次文本调用

【官方事实】Agent SDK 的 loop 过程是：

```text
接收 prompt
  -> Claude 评估当前状态
  -> 可能请求一个或多个 tool calls
  -> SDK 执行工具并把结果送回 Claude
  -> 重复
  -> 返回 final result
```

SDK 会在过程中产出多种 message：

| Message | 含义 |
| --- | --- |
| `SystemMessage` | session 生命周期事件，例如 init 和 compact boundary |
| `AssistantMessage` | Claude 每轮响应，可能包含文本和 tool calls |
| `UserMessage` | 工具结果或用户输入回到 Claude |
| `StreamEvent` | partial messages 开启时的流式事件 |
| `ResultMessage` | agent loop 结束，包含 final text、usage、cost、session id |

【作者推导】这就是第 1 篇和第 3 篇讲的 agentic loop 的程序化版本。区别是：本地 Claude Code 帮你管理 loop；Agent SDK 让你的应用接管 loop 的配置、消息流、权限、会话和观测。

### 4.2 工具、权限和预算要一起设计

【官方事实】Agent SDK 提供内置工具，包括 `Read`、`Write`、`Edit`、`Bash`、`Glob`、`Grep`、`WebSearch`、`WebFetch`、`Monitor`、`Agent`、`Skill`、`AskUserQuestion` 等。官方文档说明，可以通过 `allowed_tools` / `allowedTools`、`disallowed_tools` / `disallowedTools` 和 `permission_mode` / `permissionMode` 控制工具使用。

常见 permission mode：

| mode | 行为 |
| --- | --- |
| `default` | 未被 allow 覆盖的工具需要 approval callback；没有 callback 通常拒绝 |
| `acceptEdits` | 自动批准文件编辑和常见文件系统命令，其他 Bash 仍按规则处理 |
| `plan` | 只读探索并产出 plan，不编辑源码 |
| `dontAsk` | 不询问；未被 allow rules、settings 或 hooks 预批准的请求会被拒绝 |
| `bypassPermissions` | 允许所有 allowed tools 运行，只应在隔离环境中使用 |

【官方事实】在 `claude -p` 文档语境中，`dontAsk` 会拒绝不在 `permissions.allow` 或 read-only command set 中的请求；在 Agent SDK permissions 文档语境中，`dontAsk` 会把原本需要提示的请求直接拒绝，`canUseTool` 不会被调用。不要把它理解成“只要没写进 `allowed_tools` 就一定不可见”，更准确的理解是：它把未解决的权限请求变成 hard deny。

【官方事实】可以用 `max_turns` / `maxTurns` 限制 tool-use turns，也可以用 `max_budget_usd` / `maxBudgetUsd` 限制成本。触发限制时，`ResultMessage` 会返回对应 error subtype，例如 `error_max_turns` 或 `error_max_budget_usd`。

【作者推导】生产 agent 不应该只配置 prompt。最小配置应该同时包含：

```text
prompt
allowed tools
permission mode
max turns
budget
working directory / settings
output schema or result handling
```

没有预算和权限的 agent，就像没有超时和权限边界的后台任务。

### 4.3 Sessions：把连续工作变成可恢复对象

【官方事实】SDK 每次交互会创建或继续一个 session。可以从 `ResultMessage.session_id` 获取 session ID，并在之后 resume；这个字段在 success 和 error result 中都会出现。也可以更早从 init `SystemMessage` 中捕获 session id：TypeScript 中是 `SystemMessage` 的直接字段，Python 中在 `SystemMessage.data` 里。恢复 session 时，之前读过的文件、分析、对话历史和操作上下文会被恢复。SDK 也支持 fork session，以便在不修改原会话的情况下探索不同路径。

【作者推导】sessions 适合：

- 代码审查工具中的“继续分析”
- 长任务拆成多个请求
- 用户回到页面后继续同一个 agent 工作流
- 对同一上下文尝试两种方案并比较

不适合：

- 把 session 当长期数据库
- 把所有项目知识塞进历史对话
- 在没有 compaction 策略的情况下无限延长会话

持久规则仍然应该放在 CLAUDE.md、skills、settings 或外部配置中，而不是依赖某个很长的 session 永远不丢细节。

### 4.4 Custom tools：用 in-process MCP server 暴露应用能力

【官方事实】Agent SDK 支持 custom tools。官方文档说明，可以用 Python 的 `@tool` 或 TypeScript 的 `tool()` 定义工具，再用 `create_sdk_mcp_server` / `createSdkMcpServer` 包装成 in-process MCP server，并传给 `query()`。

一个 custom tool 至少包含：

| 部分 | 作用 |
| --- | --- |
| name | Claude 调用工具时使用的唯一标识 |
| description | Claude 判断何时调用的说明 |
| input schema | 参数结构 |
| handler | 实际执行逻辑 |

【官方事实】MCP tool 名称遵循：

```text
mcp__{server_name}__{tool_name}
```

例如 server 叫 `weather`，tool 叫 `get_temperature`，完整工具名就是：

```text
mcp__weather__get_temperature
```

【官方事实】custom tool 可以用 annotations 标注行为，例如 `readOnlyHint`。但官方文档也提醒：annotations 是 metadata，不是 enforcement。一个标记为 read-only 的 handler 仍然可能写磁盘，关键是 handler 本身要诚实。

【作者推导】custom tool 的真正风险不在 schema，而在 handler。Schema 只定义 Claude 传什么参数；handler 决定它能触碰数据库、网络、文件系统还是内部服务。

### 4.5 Hooks 和 subagents：把控制点嵌进应用进程

【官方事实】SDK hooks 是在 agent 生命周期关键点运行的 callback。Python 和 TypeScript 都支持的常见事件包括 `PreToolUse`、`PostToolUse`、`PostToolUseFailure`、`UserPromptSubmit`、`Stop`、`SubagentStart`、`SubagentStop`、`PreCompact`、`PermissionRequest`、`Notification` 等；另外还有一些 TypeScript-only 事件，例如 `SessionStart`、`SessionEnd`、`Setup`、`PostToolBatch`、`ConfigChange`、`WorktreeCreate`、`WorktreeRemove` 等。官方文档说明，hooks 运行在应用进程中，不在 agent 的 context window 里，因此不会消耗 context；`PreToolUse` 可以拒绝工具调用，Claude 会收到拒绝结果。

适合 hooks 做的事：

- 记录审计日志
- 阻止危险 Bash 命令
- 处理 `PostToolUseFailure`，把工具失败转成可追踪日志
- 用 `PermissionRequest` 接入自定义审批流程
- 用 `Notification` 把 agent 状态发到 Slack 或 PagerDuty
- 在 Stop 阶段校验 structured output
- 在 PreCompact 阶段归档 transcript
- 统计 subagent 开始和结束事件

最小 Python 配置示意：

```python
from claude_agent_sdk import ClaudeAgentOptions, HookMatcher

async def protect_env_files(input_data, tool_use_id, context):
    file_path = input_data["tool_input"].get("file_path", "")
    if file_path.endswith(".env"):
        return {
            "hookSpecificOutput": {
                "hookEventName": input_data["hook_event_name"],
                "permissionDecision": "deny",
                "permissionDecisionReason": "Do not modify .env files",
            }
        }
    return {}

options = ClaudeAgentOptions(
    hooks={
        "PreToolUse": [
            HookMatcher(matcher="Write|Edit", hooks=[protect_env_files])
        ]
    }
)
```

【作者推导】HookMatcher 的价值在于缩小 hook 的触发面。能匹配 `Write|Edit`，就不要让所有工具调用都进同一个 callback；能用 `Notification` 做异步状态通知，就不要把通知逻辑塞进会影响工具执行的 `PreToolUse`。

【官方事实】SDK 支持 subagents。主 agent 可以通过 `Agent` tool 把子任务交给 specialized agents。官方文档说明，subagent 有自己的上下文；subagent 内部消息带有 `parent_tool_use_id`，便于追踪属于哪次 delegation。

【作者推导】第 6 篇讲过 subagents 的核心价值：隔离上下文、拆分任务、控制工具。SDK 场景下，这个价值更明显，因为你可以把 subagent 的 tools、prompt、effort 和观测都纳入应用配置。

### 4.6 Structured output 和 observability

【官方事实】Agent SDK 支持 structured outputs。可以用 JSON Schema，或在 TypeScript 中用 Zod、Python 中用 Pydantic 定义 schema。Agent 完成多轮工具调用后，`ResultMessage` 中会包含符合 schema 的 `structured_output`。如果验证在重试限制内仍失败，会返回 error，而不是给你一段看起来像 JSON 的自由文本。

【作者推导】只要输出要进入数据库、UI、CI gate 或后续自动化，就应该优先考虑 structured output。自然语言适合人读，不适合系统接力。

【官方事实】Agent SDK 可以通过 OpenTelemetry 导出 traces、metrics 和 log events。官方文档说明，SDK 实际运行 Claude Code CLI child process，并通过本地 pipe 通信；OpenTelemetry instrumentation 在 CLI 中，SDK 负责把配置传给 CLI。Telemetry 默认关闭，需要设置 `CLAUDE_CODE_ENABLE_TELEMETRY=1` 和至少一个 exporter。

三类信号：

| 信号 | 内容 | 开启方式 |
| --- | --- | --- |
| Metrics | tokens、cost、sessions、lines of code、tool decisions 等计数 | `OTEL_METRICS_EXPORTER` |
| Log events | prompt、API request、API error、tool result 等结构化事件 | `OTEL_LOGS_EXPORTER` |
| Traces | interaction、model request、tool call、hook spans | `OTEL_TRACES_EXPORTER` + `CLAUDE_CODE_ENHANCED_TELEMETRY_BETA=1` |

> 注意：OpenTelemetry traces 相关增强追踪在官方文档中标注为 beta。本文只描述截至 2026-05-20 的官方文档状态。

【官方事实】默认 telemetry 是结构化的：duration、model name、tool name 等会记录；agent 读取和写入的内容默认不记录。若开启内容导出，需要额外 opt-in 变量。

【作者推导】生产环境里，至少要能回答四个问题：

```text
这次 agent 花了多少钱？
用了多少 turns？
调用了哪些工具？
失败发生在哪一步？
```

否则你不是在运行工程系统，只是在运行一段看不清内部状态的长提示词。

---

## 五、三条路径怎么选

可以用这张表判断：

| 问题 | 更适合 |
| --- | --- |
| 想在 PR / Issue 里让 Claude 响应 `@claude` | GitHub Actions |
| 想每个 PR 自动跑一次 Claude 检查 | GitHub Actions 或 Code Review |
| 想在现有 shell / CI 里插入一次 Claude 调用 | `claude -p` |
| 想拿机器可解析 JSON | `claude -p --output-format json` 或 Agent SDK structured output |
| 想在应用里维护会话、权限、工具、观测 | Agent SDK |
| 想定义自己的业务工具 | Agent SDK custom tools 或 MCP |
| 想托管式 PR inline review | Code Review |

决策树：

```text
触发源是 GitHub PR / Issue 吗？
  是 -> 需要托管式审查吗？
    是 -> Code Review
    否 -> GitHub Actions
  否 -> 只是脚本里跑一次吗？
    是 -> claude -p
    否 -> 需要长期服务、会话、自定义工具或观测吗？
      是 -> Agent SDK
      否 -> 先用 claude -p 验证
```

【作者推导】经验上，先用 `claude -p` 验证 prompt、tools、schema 和成本，再升级到 GitHub Actions 或 Agent SDK，会比一上来就写完整平台更稳。

---

## 六、常见坑和修正

### 坑 1：把 GitHub Actions 当成“更安全的本地环境”

CI 不是天然安全。Runner 有 token、secrets、网络和仓库权限。

修正：

- 最小化 workflow permissions
- 区分 fork PR 和内部 PR
- 写操作尽量走 PR comment 或 patch 建议
- 给 job 设置 timeout
- 给 Claude 设置 `--max-turns`

### 坑 2：在脚本里忘记 `--bare`

没有 `--bare` 时，`claude -p` 可能加载本机或项目里的 CLAUDE.md、hooks、skills、plugins、MCP 和 memory。团队里每个人跑出来可能不一样。

修正：

```bash
claude --bare -p "..." --settings ./ci-claude-settings.json
```

把需要的上下文显式传进去。

### 坑 3：用自然语言输出驱动自动化

错误做法：

```bash
claude -p "Tell me whether this PR is risky" | grep "safe"
```

修正：

```bash
claude --bare -p "Classify this PR risk" \
  --output-format json \
  --json-schema '{"type":"object","properties":{"risk":{"enum":["low","medium","high"]},"reasons":{"type":"array","items":{"type":"string"}}},"required":["risk","reasons"]}'
```

让输出从一开始就能被程序验证。

### 坑 4：不给 agent 设置轮次和成本上限

开放式任务可能一直读文件、跑测试、尝试修改。

修正：

- `claude_args` 中设置 `--max-turns`
- SDK 中设置 `max_turns` / `maxTurns`
- SDK 中设置 `max_budget_usd` / `maxBudgetUsd`
- CI 中设置 `timeout-minutes`

### 坑 5：custom tool handler 直接抛异常

handler 抛未捕获异常会中断整个 query，Claude 看不到可恢复的错误信息。

修正：捕获异常，并返回带 `isError: true` 的 tool result，让 agent loop 有机会换路径或解释失败。

### 坑 6：把 Code Review 当 merge gate

Code Review 的 check run 默认 neutral，不会直接阻止 merge。

修正：如果要 gate merge，就在自己的 CI 中解析 findings 或 check run 输出，并明确写入团队规则。

### 坑 7：在非隔离环境使用 `bypassPermissions`

`bypassPermissions` 会让 allowed tools 不再询问，适合隔离环境，不适合真实开发机或生产系统。

修正：

- 本地交互用 `default` 或 `acceptEdits`
- CI / container 中才考虑 `bypassPermissions`
- 写文件、跑 shell、访问外部服务时保留显式 allow / deny

### 坑 8：只看结果，不留过程

工程系统需要可追溯。只有一句“Claude said OK”不够。

修正：

- 保存 GitHub Actions run log
- 保存 structured output JSON
- 保存 session id
- 记录 total cost、turn count、tool calls
- 对 Agent SDK 开启 metrics / logs，必要时开启 traces

---

## 七、实践练习

【实践练习】下面的练习建议在测试仓库完成，不连接生产系统，不放真实密钥。每个练习至少保留一种可追溯产物：workflow diff、GitHub Actions run 日志、命令行输出、JSON 输出文件、SDK 代码片段、session id、trace 截图或最小复现仓库。

### 练习 1：用 `claude -p` 做一次 JSON 输出

**输入**：在一个测试仓库运行 `claude --bare -p "Summarize this repository" --output-format json`。  
**预期观察**：输出不是普通文本，而是包含 `result`、session metadata、usage 或 cost 相关字段的 JSON。  
**验证标准**：保存命令行输出到文件，并用 JSON 解析器验证它是合法 JSON。记录是否包含 session id 和成本字段。

### 练习 2：对比普通 `-p` 和 `--bare`

**输入**：分别运行一次 `claude -p "What project instructions are visible?"` 和 `claude --bare -p "What project instructions are visible?"`。  
**预期观察**：普通模式可能加载 CLAUDE.md、hooks、skills、plugins、MCP 或 memory；bare mode 只使用显式传入配置。  
**验证标准**：保存两次输出，标注哪些上下文来源在 bare mode 中消失。若没有差异，记录当前仓库没有相关配置。

### 练习 3：用 JSON Schema 约束输出

**输入**：让 Claude 分析 `git diff`，输出 `{ "risk": "low|medium|high", "reasons": [] }` 结构，并使用 `--json-schema`。  
**预期观察**：输出进入 `structured_output` 字段，且能被 schema 校验。  
**验证标准**：保存 schema、命令、JSON 输出和解析结果。若 validation 失败，记录失败信息。

### 练习 4：观察 `stream-json` 事件流

**输入**：运行一次 `claude -p "Explain recursion" --output-format stream-json --verbose --include-partial-messages`，并用 `jq` 或日志文件保存输出。  
**预期观察**：输出是 newline-delimited JSON，能看到 `stream_event`、partial message 或系统事件。  
**验证标准**：保存原始输出，至少标注一条文本增量事件和一条 system / metadata 事件。

### 练习 5：配置最小 GitHub Actions

**输入**：在测试仓库添加一个只读 PR 检查 workflow，使用 `anthropics/claude-code-action@v1`，限制 workflow permissions，并设置 `timeout-minutes` 和 `--max-turns`。  
**预期观察**：PR 触发后，GitHub Actions run 能启动 Claude，并根据 prompt 输出评论、日志或检查结果。  
**验证标准**：保存 workflow diff、GitHub Actions run URL、权限配置截图或日志摘要。确认 API key 来自 GitHub Secrets，而不是写在 workflow 中；如果允许 Bash，记录是否使用了 `Bash(git diff *)` 这类 scoped rule。

### 练习 6：设计 Code Review 触发策略

**输入**：写一份 markdown 方案，说明某个团队仓库是否使用 Code Review、使用 PR 创建触发、每次 push 触发还是 manual mode。  
**预期观察**：方案能区分 Code Review 和 GitHub Actions，不把 research preview 功能当成稳定 merge gate。  
**验证标准**：保存方案，必须包含触发方式、成本控制、是否解析 check run、如何处理 false positives。

### 练习 7：写一个只读 Agent SDK agent

**输入**：用 Python 或 TypeScript 写一个最小 SDK 程序，只允许 `Read`、`Glob`、`Grep`，让它总结仓库中的待办注释。  
**预期观察**：agent 能读取和搜索文件，但不能编辑文件或运行任意 Bash。  
**验证标准**：保存代码、运行输出、session id、ResultMessage subtype、total cost 或 usage。记录 session id 是从 init `SystemMessage` 还是 `ResultMessage` 获取的；如果请求它修改文件，记录会发生什么。

### 练习 8：给 SDK agent 加一个观测点

**输入**：给练习 7 的 agent 增加一种观测方式：读取 `ResultMessage` 中的 cost / turns、配置一个 `Notification` hook，或配置 OpenTelemetry metrics / logs。  
**预期观察**：运行结束后，不只知道结果，还能知道成本、轮次或工具调用过程。  
**验证标准**：保存输出或观测截图。若使用 HookMatcher，保存 hook 配置片段；若使用 traces，标注它在官方文档中是 beta，并记录开启的环境变量。

---

## 实践复盘建议

做完练习后，建议按三类整理：

- 与官方文档一致：`@claude` 是否按预期触发？`claude -p` 是否支持 JSON / stream-json？token 级流式输出是否使用了 `--verbose --include-partial-messages`？`--bare` 是否跳过本地配置？Agent SDK 是否返回 `SystemMessage` / `ResultMessage`、session id、usage / cost？
- 与预期不同：是 workflow permissions 不足、secret 未配置、fork PR 拿不到 secret、`--bare` 缺少必要上下文、schema 太严格、`allowedTools` 规则太宽或太窄、tool 权限不足，还是 max turns 太小？
- 文档没有覆盖：保存最小复现仓库、workflow run、命令输出、JSON、SDK 代码和日志，后续 Claude Code 版本升级后再复测。

这篇文章的核心结论可以压成一句话：

```text
GitHub Actions 让 Claude 进入仓库事件，
Headless 让 Claude 进入脚本，
Agent SDK 让 Claude 进入应用；
真正的工程化，不是自动化更多，而是边界更清楚。
```

---

## 参考资料

### 官方主参考

- [Claude Code GitHub Actions](https://code.claude.com/docs/en/github-actions)
- [Code Review](https://code.claude.com/docs/en/code-review)
- [Run Claude Code programmatically](https://code.claude.com/docs/en/headless)
- [Agent SDK overview](https://code.claude.com/docs/en/agent-sdk/overview)
- [How the agent loop works](https://code.claude.com/docs/en/agent-sdk/agent-loop)
- [Get structured output from agents](https://code.claude.com/docs/en/agent-sdk/structured-outputs)
- [Observability with OpenTelemetry](https://code.claude.com/docs/en/agent-sdk/observability)
- [Give Claude custom tools](https://code.claude.com/docs/en/agent-sdk/custom-tools)

### GitHub / 社区补充

- [anthropics/claude-code-action](https://github.com/anthropics/claude-code-action)
- [anthropics/claude-agent-sdk-python](https://github.com/anthropics/claude-agent-sdk-python)
- [anthropics/claude-code-security-review](https://github.com/anthropics/claude-code-security-review)

GitHub 和社区仓库更适合观察 workflow 示例、SDK 代码结构和安全审查实践，不用于替代官方文档对 Claude Code 行为的定义。凡是社区经验与官方文档冲突，以官方文档和本地实践观察为准。
