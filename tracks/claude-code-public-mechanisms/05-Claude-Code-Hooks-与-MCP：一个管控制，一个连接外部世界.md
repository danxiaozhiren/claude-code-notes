# Hooks 与 MCP：一个管控制，一个连接外部世界

> 文档核查日期：2026-05-19  
> 作者：wt  
> 写作初衷：在领导的鼓励与倡导下，围绕 Claude Code 做一次面向底层原理的系统学习，把学习过程中的理解、积累和实践经验沉淀下来，并以文章形式分享给更多同事和读者。  
> 适合读者：团队负责人、平台开发者、进阶个人开发者  
> 本文定位：Claude Code 系列第 5 篇，承接第 4 篇的权限与安全边界，解释 Hooks 和 MCP 这两类扩展为什么不能混为一谈。  
> 实践建议：本文末尾提供动手练习，建议读者用无真实密钥的测试仓库验证 hook 触发、MCP 连接、权限提示和 context 成本。

---

## 先把边界说清楚

前四篇已经建立了 Claude Code 的总模型、context 装载、工具系统和安全边界。这篇承接第 4 篇，聚焦两个很容易混在一起的扩展能力：

```text
Hooks：在 Claude Code 生命周期节点上自动运行控制逻辑。
MCP：把 Claude Code 连接到外部系统、工具、数据源。
```

它们都能扩展 Claude Code，但解决的问题完全不同。

本文关注：

- Hooks 生命周期事件：PreToolUse、PostToolUse、Stop、InstructionsLoaded、PreCompact、PostCompact 等
- Hooks 为什么不等同于提示词
- command / HTTP / prompt / agent / MCP tool hooks 的角色
- Hooks 的输入、输出、exit code 和决策控制
- MCP server、tools、prompts、resources
- MCP prompts 如何变成 slash commands
- MCP 的权限、安全、elicitation、context 成本和输出限制
- MCP + Skills + Hooks 的组合模式

不展开的内容：

- Subagents、Agent Teams、Worktrees：放到第 6 篇
- Skills、Plugins、Marketplaces 的标准化分发：放到第 7 篇
- GitHub Actions、Headless、Agent SDK：放到第 8 篇

本文只在容易混淆的地方标注来源类型：

- 【官方事实】：来自 Claude Code 官方文档或官方仓库。
- 【官方 GitHub】：来自 Anthropic 官方 GitHub 仓库。
- 【作者推导】：基于官方事实整理出的机制解释和工程判断。
- 【实践练习】：读者可以在自己的环境中复现观察的部分。

---

## 一、核心认知：Hooks 管“什么时候必须发生”，MCP 管“能连接到哪里”

先用一句话区分：

```text
Hooks 是控制层。
MCP 是连接层。
```

【官方事实】Claude Code 官方扩展概览把 Hooks 描述为在生命周期事件触发时运行脚本、HTTP 请求、prompt 或 subagent 的机制；把 MCP 描述为连接外部服务和工具的机制。

【官方事实】Hooks 文档说，hooks 是用户定义的命令、HTTP endpoint 或 LLM prompt，会在 Claude Code 生命周期的特定点自动执行，用来格式化代码、发通知、验证命令、执行项目规则。

【官方事实】MCP 文档说，Claude Code 可以通过 Model Context Protocol 连接外部工具和数据源，让 Claude 访问 issue tracker、监控平台、数据库、设计工具、邮件系统、浏览器等。

【作者推导】两者差异可以压成这张表：

| 问题 | Hooks | MCP |
| --- | --- | --- |
| 它解决什么 | 每次到某个节点，都执行确定动作 | 让 Claude 获得外部工具和数据 |
| 谁触发 | Claude Code 生命周期事件 | Claude 选择调用某个外部工具，或用户执行 MCP prompt |
| 典型动作 | 编辑后格式化、命令前拦截、任务结束前检查 | 查 Jira、读 Sentry、查询数据库、发 Slack |
| 安全风险 | hook 命令以用户权限运行，可能误删/误改 | 外部系统权限、数据外泄、prompt injection |
| context 成本 | 默认不进 context，除非 hook 返回内容 | 工具名启动时加载，schema 按需加载，输出会进 context |
| 最适合 | 必须发生的自动化和管控 | Claude 原本看不见的外部世界 |

所以：

- “每次 Edit 后都跑 formatter”是 hook。
- “让 Claude 查询数据库”是 MCP。
- “Claude 用数据库 MCP 查数据前，hook 记录审计日志”是 Hook + MCP。
- “把数据库使用规范写成 skill，让 Claude 正确使用数据库 MCP”是 Skill + MCP。

这三类东西不要互相替代。

---

## 二、Hooks 为什么不等同于提示词

很多人会问：

```text
我在 CLAUDE.md 写“每次改完代码都运行测试”，不就行了吗？
为什么还要 hook？
```

答案是：提示词影响模型意图，hook 绑定运行时事件。

【官方事实】Hooks 会在 Claude Code 生命周期的特定事件触发。事件触发后，Claude Code 把 JSON 输入传给 hook handler，handler 可以检查输入、执行动作，并通过 exit code 或 JSON 输出影响后续行为。

【作者推导】这就是第 4 篇安全边界逻辑的延伸：

```text
提示词：告诉模型“你应该做什么”
Hook：告诉 harness“到这个节点就必须跑什么”
```

例如：

```text
CLAUDE.md：改完代码后请格式化。
PostToolUse hook：只要 Edit / Write 成功，就运行 formatter。
```

第一种依赖模型记得、理解并选择执行。第二种是事件驱动，只要条件匹配就执行。

这不代表 hooks 总是比提示词好。它们负责的层不同：

| 场景 | 更适合 |
| --- | --- |
| 项目约定、架构背景、命名规则 | CLAUDE.md / rules |
| 可复用操作流程、检查清单 | Skill |
| 每次工具调用前后都要执行的控制 | Hook |
| 访问外部系统和工具 | MCP |
| 分发一整套配置 | Plugin |

【作者推导】判断一个规则要不要变成 hook，可以问三个问题：

```text
它是否必须每次发生？
它是否可以用确定性脚本判断？
它是否不应该依赖 Claude 自己想起来？
```

如果答案都是是，就适合 hook。

---

## 三、Hooks 生命周期：在哪些节点能插手

【官方事实】Hooks 按生命周期事件触发。官方参考文档把事件分成几类：session 级、turn 级、tool-use 级，以及配置、文件、compaction、MCP elicitation 等异步事件。

这篇不罗列所有事件，只讲最影响工程实践的几个。

| 事件 | 触发时机 | 常见用途 |
| --- | --- | --- |
| `SessionStart` | 会话开始或恢复 | 注入环境提醒、加载动态上下文 |
| `UserPromptSubmit` | 用户 prompt 交给 Claude 前 | 检查敏感请求、补充上下文 |
| `PreToolUse` | 工具调用执行前 | 拦截危险命令、限制写入、要求确认 |
| `PermissionRequest` | 权限弹窗出现时 | 自动批准极窄范围的提示 |
| `PostToolUse` | 工具调用成功后 | 格式化、记录审计、触发检查 |
| `PostToolUseFailure` | 工具调用失败后 | 记录失败、提供诊断 |
| `PostToolBatch` | 一批并行工具调用结束后 | 在下一次模型调用前做整体检查 |
| `Stop` | Claude 准备结束回复时 | 检查任务是否完成、阻止过早停止 |
| `InstructionsLoaded` | CLAUDE.md 或 rules 被加载时 | 观察规则加载、审计上下文来源 |
| `PreCompact` | compact 前 | 记录状态、必要时阻止 compact |
| `PostCompact` | compact 后 | 记录 compact summary、更新外部状态 |
| `ConfigChange` | 配置文件变化时 | 审计配置改动、阻止未授权修改 |
| `Elicitation` | MCP server 请求用户输入时 | 审查外部 server 要求的输入 |

【官方事实】官方文档说明，`PreToolUse` 可以阻止工具调用；`PostToolUse` 发生在工具成功之后，所以不能阻止已经发生的工具调用；`Stop` 可以阻止 Claude 停止，让它继续工作；`InstructionsLoaded` 没有决策控制，主要用于观测；`PostCompact` 不能影响 compact 结果，只能做后续动作。

【作者推导】这里的机制差异很重要：

```text
Pre：动作还没发生，可以阻止。
Post：动作已经发生，只能补救、记录、提示。
Stop：回复要结束，可以要求继续。
Compact：摘要过程前后，可以审计或补充，但不要滥用阻止。
```

如果你要防止危险行为，优先用 `PreToolUse`。如果你只是想格式化、记录、通知，用 `PostToolUse`。如果你要防止 Claude “没做完就收工”，用 `Stop`。

---

## 四、Hooks 如何工作：事件、matcher、handler、输出

一个 hook 配置通常有三层：

```text
事件：什么时候触发
matcher：只匹配哪些情况
handler：触发后运行什么
```

例如：

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Edit|Write",
        "hooks": [
          {
            "type": "command",
            "command": "${CLAUDE_PROJECT_DIR}/.claude/hooks/log-edit.sh",
            "args": []
          }
        ]
      }
    ]
  }
}
```

【官方事实】Hooks 可以定义在多个位置：用户 settings、项目 `.claude/settings.json`、本地 `.claude/settings.local.json`、managed policy settings、插件的 `hooks/hooks.json`，也可以定义在 skills 或 agents frontmatter 中。`/hooks` 菜单是只读浏览器，用来查看事件、matcher、来源文件和 handler 细节。

### 4.1 matcher 解决“只在什么时候跑”

【官方事实】`PreToolUse`、`PostToolUse`、`PostToolUseFailure`、`PermissionRequest`、`PermissionDenied` 这类 tool event 的 matcher 匹配工具名。例如：

```json
{
  "matcher": "Bash"
}
```

只匹配 Bash 工具。

```json
{
  "matcher": "Edit|Write"
}
```

匹配 Edit 或 Write。

【官方事实】对于 MCP tool，也可以像普通工具一样匹配，命名形态是：

```text
mcp__<server>__<tool>
```

例如：

```text
mcp__github__search_repositories
mcp__filesystem__read_file
mcp__memory__create_entities
```

如果要匹配某个 server 下的所有工具，需要写成：

```text
mcp__memory__.*
```

不能只写 `mcp__memory`。

【官方事实】一些事件不支持 matcher，例如 `Stop`、`UserPromptSubmit`、`PostToolBatch`、`CwdChanged` 等。如果给这些事件写 matcher，会被忽略。

【作者推导】这意味着 hooks 不是越多越好，而是越窄越可靠。尤其是权限相关 hook，matcher 应该尽可能窄。

不要这样：

```json
{
  "matcher": ".*"
}
```

轻易匹配所有工具。

更好的方式是：

```json
{
  "matcher": "Bash",
  "hooks": [
    {
      "type": "command",
      "if": "Bash(git push *)",
      "command": "${CLAUDE_PROJECT_DIR}/.claude/hooks/check-push.sh",
      "args": []
    }
  ]
}
```

这样只有 `git push` 相关命令才进入检查。

### 4.2 handler 解决“触发后跑什么”

【官方事实】Hooks reference 列出五类 hook handler。这里先只建立“handler 类型”的词汇表，第五节再按使用场景展开：

| handler 类型 | 做什么 | 适合场景 |
| --- | --- | --- |
| `command` | 运行 shell 命令 | 本地脚本、formatter、审计、命令校验 |
| `http` | 向 HTTP endpoint 发送事件 JSON | 团队审计服务、外部通知、集中策略服务 |
| `mcp_tool` | 调用已连接 MCP server 的工具 | 用外部扫描器或企业系统做检查 |
| `prompt` | 让模型做一次判断 | 输入足够、需要轻量判断 |
| `agent` | 启动 agent 做验证 | 需要读文件、搜索代码、跑命令的复杂判断 |

> 注意：`agent` hooks 在官方文档中标注为 experimental。本文只描述截至 2026-05-19 的观察结果，不将其视为稳定行为保证。生产工作流优先使用 command hooks。

【作者推导】handler 类型也体现了“确定性 vs 判断性”的差异：

```text
command / http / mcp_tool：偏工程执行
prompt / agent：偏模型判断
```

能用脚本确定的事情，不要交给模型判断。例如“是否命中了 `.env` 路径”“是否运行了 `rm -rf`”“是否编辑了 migrations 目录”，都适合 command hook。

需要语义判断的事情，才考虑 prompt hook。例如“这次总结是否覆盖了用户请求的三件事”。

需要实际读取代码或运行测试的验证，再考虑 agent hook。

### 4.3 输出解决“怎么影响后续行为”

【官方事实】Command hooks 通过 stdin 接收 JSON，通过 stdout、stderr 和 exit code 返回结果。exit code 0 表示成功；exit code 2 表示阻止动作；其他 exit code 多数情况下是非阻塞错误。

一个最小拦截例子：

```bash
#!/bin/bash
input=$(cat)
command=$(echo "$input" | jq -r '.tool_input.command // ""')

if echo "$command" | grep -q "rm -rf"; then
  echo "Blocked: rm -rf is not allowed" >&2
  exit 2
fi

exit 0
```

【官方事实】对于 `PreToolUse`，exit 2 会阻止工具调用；对于 `PostToolUse`，工具已经执行完成，exit 2 不能阻止已经发生的动作，只能把 stderr 反馈给 Claude。不同事件的 exit code 2 行为不同。

【官方事实】如果需要更细控制，可以 exit 0 并输出 JSON，例如在 `PreToolUse` 中返回 `permissionDecision: "deny" / "allow" / "ask"`。不过 hook 返回 `allow` 不会覆盖 deny 规则；deny / ask 权限规则仍然优先。

【作者推导】这条规则和第 4 篇一致：hook 可以补充控制，但不能越过权限底线。

```text
permissions.deny > hook allow
managed deny > hook allow
```

---

## 五、Hooks 的典型用法：不是炫技，而是把重复控制固化

第四节按 handler 类型解释了“hook 能跑什么”。这一节换一个角度，按使用场景解释“什么时候应该用 hook”。

Hooks 最适合三类事情。

### 5.1 后置自动化：每次编辑后都做

例如每次 Edit / Write 后记录日志：

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Edit|Write",
        "hooks": [
          {
            "type": "command",
            "command": "${CLAUDE_PROJECT_DIR}/.claude/hooks/log-edit.sh",
            "args": []
          }
        ]
      }
    ]
  }
}
```

可以做：

- 格式化
- 记录 diff 摘要
- 触发轻量 lint
- 通知团队
- 写审计日志

【作者推导】PostToolUse 适合“补强”和“记录”，不适合“预防”。如果真的要预防，动作发生前就要用 PreToolUse。

### 5.2 前置拦截：危险动作前先挡住

例如阻止危险 Bash：

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "if": "Bash(rm *)",
            "command": "${CLAUDE_PROJECT_DIR}/.claude/hooks/block-rm.sh",
            "args": []
          }
        ]
      }
    ]
  }
}
```

可以做：

- 禁止 `rm -rf`
- 禁止 `git push main`
- 禁止 `npm publish`
- 禁止修改迁移脚本
- 禁止调用某些 MCP 写入工具

【作者推导】PreToolUse 和 permissions 的关系可以这样理解：

```text
permissions：静态边界，适合简单规则
PreToolUse hook：动态边界，适合需要脚本判断的规则
```

能用 `permissions.deny` 写清楚的，就先写 deny；只有当规则依赖上下文、路径组合、分支名、任务编号、文件内容时，再用 hook。

### 5.3 收尾检查：别让 Claude 过早停止

Stop hook 在 Claude 准备结束时触发。

适合：

- 检查任务清单是否完成
- 检查是否有未运行测试
- 检查是否修改了文件却没有总结 diff
- 检查是否遗漏用户要求的输出

【官方事实】Stop hook 可以阻止 Claude 停止，让它继续对话；官方也说明 Stop hook 连续阻止太多次会触发上限，避免无限循环。

【作者推导】Stop hook 很有用，也容易滥用。不要把“每次都必须重新跑全量测试”放进 Stop hook，否则会把小改动也变成沉重流程。更好的做法是：只检查“该不该继续”，真正执行验证仍然交给 Claude 或明确脚本。

---

## 六、Hooks 的安全边界：它们本身也是代码

【官方事实】Hooks reference 明确提醒：command hooks 以当前系统用户权限运行。它们可以修改、删除、访问当前用户能访问的文件。

这句话要当成安全底线。

【作者推导】Hook 不是安全魔法。一个写坏的 hook，本身就是新的风险入口：

| 风险 | 例子 | 修正 |
| --- | --- | --- |
| 输入未校验 | 直接把 `tool_input.file_path` 拼进 shell 命令 | JSON 解析后校验路径，禁止 `..` |
| 变量未引用 | 用 `$FILE` 而不是 `"$FILE"` | 始终 quote shell 变量 |
| 路径不稳定 | 相对路径随 cwd 变化 | 使用 `${CLAUDE_PROJECT_DIR}` 或绝对路径 |
| 依赖缺失 | hook 里用 `jq`，机器上没装 | 练习前检查依赖，或用 Node/Python |
| 输出污染 | shell profile 自动 echo，破坏 JSON 输出 | 非交互 shell 不输出杂音 |
| 规则过宽 | PermissionRequest 自动 approve 所有提示 | matcher 和 `if` 尽量窄 |

【官方事实】官方文档建议使用绝对路径、校验输入、引用变量、避免敏感文件、调试时查看 debug log。Windows 上可以通过 `"shell": "powershell"` 指定 PowerShell。

【作者推导】团队里推荐把重要 hook 当成普通生产脚本管理：

- 放进仓库
- 有代码 review
- 有最小测试样例
- 有失败时的降级行为
- 有日志
- 不在 hook 里硬编码密钥
- 重要 hook 用 managed settings 或 plugin 分发

如果 hook 的安全级别比它要防的风险还低，那它就不是护栏，而是新的缺口。

---

## 七、MCP 是什么：让 Claude Code 看到外部系统

MCP 的全称是 Model Context Protocol。

【官方事实】Claude Code 可以通过 MCP 连接外部工具、数据库和 API。官方举的例子包括 issue tracker、监控数据、PostgreSQL 数据库、Figma / Slack / Gmail 等外部系统。

【作者推导】如果说内置工具让 Claude Code 进入本地工程环境，那么 MCP 让 Claude Code 进入组织系统：

```text
本地项目
  + 内置工具：Read / Edit / Bash / Grep / LSP
  + MCP：GitHub / Jira / Sentry / DB / Slack / Browser / 内部平台
```

这带来两个变化：

- Claude 不再只根据你贴进来的内容判断，而是能直接查询外部系统。
- Claude 的行动边界不再只在本机，还可能影响真实业务系统。

这就是为什么第 4 篇先讲了 MCP 安全风险：MCP 的能力越强，权限设计越要先行。

---

## 八、MCP server、tools、prompts、resources：不要只盯着 tools

很多人一提 MCP 就只想到 tools。其实 MCP server 暴露的能力不止一种。

### 8.1 Server：能力提供者

MCP server 是连接外部系统的服务。

【官方事实】Claude Code 支持多种 MCP transport。HTTP 是连接远程 server 的推荐方式；SSE 已被官方标注为 deprecated；stdio server 作为本地进程运行，适合需要本机访问或自定义脚本的场景。

常见 server 来源：

- 手动 `claude mcp add`
- 项目 `.mcp.json`
- 用户 `~/.claude.json`
- 插件提供的 `.mcp.json`
- Claude.ai connectors

【官方事实】当同名 server 出现在多个 scope 时，Claude Code 按优先级只连接一个。官方文档列出的顺序是 local、project、user、plugin-provided servers、claude.ai connectors。

【作者推导】这意味着 MCP 不是只要“能连上”就完事。你还要知道这个 server 从哪里来、是否被更高优先级配置覆盖、是否带有真实凭据。

### 8.2 Tools：Claude 可以调用的外部动作

MCP tools 是 Claude 可以调用的外部能力。

例如：

```text
mcp__github__search_repositories
mcp__sentry__list_issues
mcp__db__query
mcp__browser__navigate
```

【官方事实】MCP tools 可以通过 permissions 规则控制。例如 `mcp__server__tool` 匹配指定 server 的指定 tool，`mcp__server__.*` 匹配某个 server 的全部 tools。

【作者推导】MCP tool 权限至少要分读写：

| 类型 | 例子 | 建议 |
| --- | --- | --- |
| 只读查询 | 查 issue、读监控、读文档 | 可 allow 或 ask，视敏感程度 |
| 写外部系统 | 建 issue、发消息、改配置 | 默认 ask |
| 数据库读取 | 查 schema、查样例数据 | 只用 readonly 账号，限制范围 |
| 数据库写入 | update / delete / DDL | 默认 deny |
| 浏览器自动化 | 打开页面、点击、提交表单 | 对登录态页面保持 ask |

一个最小示意：

```json
{
  "permissions": {
    "allow": [
      "mcp__docs__search"
    ],
    "ask": [
      "mcp__github__create_issue",
      "mcp__browser__navigate"
    ],
    "deny": [
      "mcp__db__write",
      "mcp__.*__delete.*"
    ]
  }
}
```

这不是通用模板，而是提醒：MCP tool 也要进入 allow / ask / deny，不要因为它是“外部工具”就跳过权限设计。

### 8.3 Prompts：MCP 也能提供 slash commands

【官方事实】MCP server 可以暴露 prompts，这些 prompts 会在 Claude Code 中变成命令，格式类似：

```text
/mcp__servername__promptname
```

例如：

```text
/mcp__github__list_prs
/mcp__github__pr_review 456
/mcp__jira__create_issue "Bug in login flow" high
```

【官方事实】MCP prompts 是从连接的 server 动态发现的，执行结果会直接注入对话。

【作者推导】MCP prompt 和 skill 命令看起来都像 slash command，但含义不同：

| 项 | MCP prompt | Skill command |
| --- | --- | --- |
| 来源 | 外部 MCP server | 本地 / 插件中的 skill |
| 主要作用 | 让外部系统提供任务模板或操作入口 | 复用本地知识、流程、工作流 |
| 风险 | 外部 server 提供的 prompt 内容和参数 | 本地流程可能触发工具和改动 |
| 管理重点 | server 信任、权限、认证 | skill 描述、触发方式、上下文成本 |

如果某个外部系统既提供 tool 又提供 prompt，你要同时审查：

- prompt 会把什么内容注入对话
- tool 会对外部系统做什么
- 参数是否可能包含敏感信息

### 8.4 Resources：外部系统里的可引用内容

【官方事实】MCP resources 可以被 `@` mention 引用。被引用后，Claude Code 会获取资源并作为附件加入对话。资源可以是文本、JSON、结构化数据等。

【作者推导】Resources 适合“读一个外部对象”，例如数据库 schema、设计文档、知识库页面。它们的风险和文件读取很像：内容进入 context 后，就会影响模型判断，也可能包含 prompt injection。

所以，对外部 resources 的态度应该接近第 4 篇里的“不可信内容”：

```text
能读，不代表可信；
能进入 context，不代表它有指令优先级。
```

## 九、MCP 的 context 成本：不是全量塞进窗口

第 2 篇和第 3 篇都讲过 context 成本。MCP 也绕不开这个问题。

【官方事实】官方扩展概览说明：MCP servers 在 session start 时加载 tool names，完整 JSON schemas 按需加载。MCP Tool Search 默认启用，会推迟 tool definitions，Claude 需要时再搜索相关工具。只有实际使用的工具进入 context。

【官方事实】MCP 文档说明，Tool Search 默认让 MCP 工具 schema 延迟加载，启动时只加载工具名，以降低 context 占用。用户也可以配置工具搜索策略。

【作者推导】这解释了一个常见疑问：

```text
为什么我连了很多 MCP server，/context 没有立刻爆炸？
```

因为默认不是把所有 tool schema 都塞进 context，而是先让 Claude 知道“有哪些工具名字”，需要时再搜索和加载。

但这不等于 MCP 没有 context 成本：

- tool 名称和 server 描述仍然有成本
- 实际调用的 tool schema 会进入 context
- tool 输出会进入 context 或以文件引用形式保留
- resources 和 prompts 注入内容会增加 context

【官方事实】MCP 文档还说明，MCP tool 输出超过一定规模会出现警告；默认最大输出限制是 25,000 tokens，可通过 `MAX_MCP_OUTPUT_TOKENS` 调整。server 作者可以通过 `_meta["anthropic/maxResultSizeChars"]` 为特定工具声明更高的文本输出阈值。

【作者推导】MCP 的 context 策略和第 3 篇的大输出原则一致：

```text
不要让外部系统一次返回一片海。
让 MCP tool 支持分页、过滤、摘要、按需读取。
```

特别是数据库、日志、监控、知识库这类 MCP。它们很容易把大量数据带进上下文，读者应该主动要求 Claude 先筛选，再展开。

---

## 十、MCP 权限与安全：外部能力要先问边界

【官方事实】MCP 文档提醒，在连接 MCP server 前要确认信任，尤其是会获取外部内容的 server，因为外部内容可能带来 prompt injection 风险。

【官方事实】Claude Code 支持远程 MCP OAuth。认证 token 会被安全存储并自动刷新，`/mcp` 可用于完成认证、查看和管理 server 状态。

【官方事实】组织可以用 `managed-mcp.json` 完全控制可用 MCP server，也可以用 managed settings 中的 `allowedMcpServers` / `deniedMcpServers` 做 allowlist / denylist。限制可以按 server name、stdio command、remote URL pattern 三种方式写。

【作者推导】MCP 安全要看三层：

| 层 | 要问什么 | 控制方式 |
| --- | --- | --- |
| server | 这个 server 是否可信，来源是否可控 | managed-mcp、allowlist、denylist |
| tool | 这个 tool 能读还是能写，能影响哪个系统 | permissions allow / ask / deny |
| output | 返回内容是否可信，是否过大，是否含 injection | prompt injection 防护、输出限制、分页 |

【官方事实】MCP server 还可以在工具调用过程中请求用户输入。Claude Code 会显示交互对话框或打开 URL 流程，并把用户响应传回 server。Hooks 里也有 `Elicitation` 和 `ElicitationResult` 事件，可用于拦截或审查这类请求。

【作者推导】Elicitation 更适合放在安全视角理解，而不是和 tools / prompts / resources 同等看待。因为这时外部 server 不只是返回内容，而是在向用户索取额外输入。团队使用时，要明确：

- server 能不能请求凭据
- 请求输入的字段是否合理
- 响应是否会被发回外部系统
- 是否需要 hook 或 managed policy 审查

一个团队级判断流程：

```text
要不要接这个 MCP server？
  -> 它背后有没有真实凭据？
  -> 它能不能写外部系统？
  -> 它返回的内容是否来自不可信用户？
  -> 它能不能按只读 / 写入分 tool 控制？
  -> 它是否支持最小 OAuth scope 或 readonly token？
  -> 它是否应该由 managed settings 统一下发？
```

【作者推导】最危险的 MCP 不是“看起来危险的工具”，而是“名字像只读，实际能产生外部副作用”的工具。比如一个 `sync`、`update_status`、`send_message`、`run_query`，都需要看 server 文档和权限范围，而不是只看名字。

---

## 十一、MCP + Skills + Hooks：三者组合才是工程化

官方扩展概览里有一个很实用的观点：每个扩展解决不同问题，可以组合。

【官方事实】MCP 提供外部连接；Skills 提供知识和工作流；Hooks 在事件上自动触发；Plugins 可以把这些能力打包分发。

【作者推导】可以把三者想成：

```text
MCP：能力
Skill：用法
Hook：管控
```

例如数据库场景：

| 组件 | 负责什么 |
| --- | --- |
| MCP | 提供 readonly query、schema read、sample data read |
| Skill | 说明库表含义、查询规范、禁止扫全表、常用分析模板 |
| Hook | 拦截写入工具、记录 query、限制敏感表、审计大输出 |

再比如发布场景：

| 组件 | 负责什么 |
| --- | --- |
| MCP | 连接 GitHub、Slack、监控平台 |
| Skill | 提供发布 checklist、回滚步骤、通知模板 |
| Hook | 在任务结束前检查测试、变更摘要、是否有审批记录 |

再比如安全审查场景：

| 组件 | 负责什么 |
| --- | --- |
| MCP | 连接 Sentry、GitHub、漏洞扫描系统 |
| Skill | 提供审查维度和报告格式 |
| Hook | 每次 Edit 后调用安全扫描或记录改动 |

【作者推导】这也是第 5 篇和后续第 7 篇的分界：

- 本文讲组合模式的架构判断。
- 第 7 篇会讲如何把 skills、hooks、MCP、settings 打包成 plugin，并通过 marketplace 分发。

---

## 十二、常见坑和修正

### 坑 1：把 hook 当成提示词增强

错误认知：

```text
hook 就是另一种告诉 Claude 该怎么做的方式。
```

修正：

```text
hook 是运行时事件处理器。
它不只是影响 Claude 想什么，还能在特定节点执行脚本、阻止动作、记录审计。
```

### 坑 2：在 PostToolUse 里做预防

错误认知：

```text
我在 PostToolUse 里发现危险命令，就能阻止它。
```

修正：

```text
PostToolUse 发生在工具成功之后。
要阻止危险动作，用 PreToolUse 或权限 deny。
```

### 坑 3：hook 规则写得太宽

错误配置：

```json
{
  "matcher": ".*"
}
```

或者自动批准所有 `PermissionRequest`。

修正：

- matcher 写窄
- `if` 写窄
- 对自动批准保持极小范围
- deny 仍放在 permissions 或 managed settings

### 坑 4：忘记 hook 自身有系统权限

hook 不是运行在模型里，而是运行在你的机器上。它能做的事情，取决于当前系统用户权限。

修正：

- 不信任 hook 输入
- 不硬编码密钥
- 不拼接未经校验的路径
- 用绝对路径
- 用测试 JSON 单独跑 hook 脚本

### 坑 5：把 MCP 当成普通内置工具

MCP 背后可能是真实数据库、真实 issue tracker、真实浏览器登录态。

修正：

- 先审 server 来源
- 再审 tool 权限
- 最后审输出内容和 context 成本
- 对写入型 tool 默认 ask 或 deny

### 坑 6：只连接 MCP，不写使用规范

连接数据库 MCP 后，如果没有 skill 或 CLAUDE.md 约束，Claude 可能不知道你的查询边界、业务口径、敏感表、性能要求。

修正：

- MCP 提供连接
- Skill 提供使用规范
- Hook 提供审计和拦截

### 坑 7：忽略 MCP prompts

MCP prompts 会变成 slash commands，执行结果会进入对话。它们不是普通本地命令，也需要审查来源。

修正：

- 用 `/` 查看 MCP prompts
- 关注 prompt 参数
- 关注 prompt 返回内容
- 对外部 server 保持信任边界

---

## 十三、实践练习

【实践练习】下面的练习建议在测试仓库完成，不放真实密钥，不连接生产系统。每个练习至少保留一种可追溯产物：配置文件 diff、hook 脚本、`/hooks` 截图、`/mcp` 输出、命令行输出、`/context` 对比或最小复现仓库。

### 练习 1：写一个 PostToolUse hook 记录编辑

**输入**：在 `.claude/settings.json` 中添加一个 `PostToolUse` hook，matcher 为 `Edit|Write`，调用 `.claude/hooks/log-edit.sh`，把 `tool_input.file_path` 和时间写入 `.claude/hook-log.jsonl`。  
**预期观察**：每次 Claude Code 成功 Edit 或 Write 后，日志文件增加一行；`/hooks` 能看到该 hook 注册在 `PostToolUse` 下。  
**验证标准**：保存 `.claude/settings.json` diff、hook 脚本、`/hooks` 截图和日志文件片段。日志中不应包含敏感文件内容，只记录路径和事件信息。

### 练习 2：写一个 PreToolUse hook 拦截危险 Bash

**输入**：写一个 `PreToolUse` hook，matcher 为 `Bash`，当命令包含 `rm -rf`、`npm publish` 或 `git push origin main` 时返回 exit 2 或 JSON deny。  
**预期观察**：Claude Code 在执行匹配命令前被阻止，并把拦截原因反馈给 Claude；不匹配的安全命令可以继续执行。  
**验证标准**：保存 hook 脚本、手工输入 JSON 测试输出、Claude Code 会话截图或 debug log。记录是 exit 2 阻止，还是 JSON `permissionDecision: "deny"` 阻止。

### 练习 3：观察 Stop hook 的收尾检查

**输入**：配置一个轻量 `Stop` hook，只检查当前任务是否还有未完成标记，例如读取一个测试用的 `.claude/task-check.txt`，如果存在 `PENDING` 就阻止停止并输出原因。  
**预期观察**：Claude 准备结束时，Stop hook 可以让它继续处理未完成项；当 `PENDING` 被清除后，允许结束。  
**验证标准**：保存配置 diff、测试文件、会话截图。注意记录是否出现连续阻止过多导致的保护提示，避免写出无限循环 hook。

### 练习 4：连接一个无敏感凭据的最小 MCP server

**输入**：选择一个无真实业务权限的 MCP server，例如本地 demo server、只读文档检索 server，或团队内部的测试 server。用 `claude mcp add` 添加后，在 Claude Code 中运行 `/mcp`。  
**预期观察**：`/mcp` 能显示 server 状态和工具数量；如果 server 需要认证，会进入认证流程；如果连接失败，会显示 pending、failed 或其他状态。  
**验证标准**：保存 `claude mcp list` 或 `/mcp` 输出、`.mcp.json` 或配置 diff。不要提交 token、header、API key。

### 练习 5：观察 MCP tool 的权限和 context 成本

**输入**：对一个 MCP tool 分别配置 allow / ask / deny，执行一次只读查询，并在调用前后运行 `/context`。如果 server 提供 prompt，也用 `/` 找到 `/mcp__servername__promptname` 形式的命令并执行一次无副作用 prompt。  
**预期观察**：MCP tool 受 permissions 控制；调用后 context 里能看到工具 schema 或输出带来的变化；MCP prompt 的结果会注入对话。  
**验证标准**：保存 `/permissions`、`/context`、`/mcp` 输出和会话截图。记录 tool search 是否让 idle MCP server 的 context 成本保持较低。

### 练习 6：设计一个 MCP + Skill + Hook 组合方案

**输入**：不用真实接系统，只写一份设计草案：假设要把 Claude Code 接入一个只读数据库 MCP，应该配什么 skill，配什么 hook，配什么 permissions。  
**预期观察**：方案能清楚区分：MCP 提供什么能力，skill 提供什么使用知识，hook 负责什么审计或拦截。  
**验证标准**：保存一份 markdown 设计文档，至少包含 server 来源、tool 权限、敏感表策略、大输出策略、审计 hook、实践验证步骤。

---

## 实践复盘建议

做完练习后，建议按三类整理：

- 与官方文档一致：`PreToolUse` 是否能阻止工具调用？`PostToolUse` 是否只能在事后补强？`/hooks` 是否能显示 hook 来源？`/mcp` 是否能显示 server 状态和 tool 数量？MCP prompts 是否以 `/mcp__server__prompt` 形式出现？
- 与预期不同：是 matcher 写错、事件不支持 matcher、hook 脚本没有执行权限、`jq` 缺失、JSON 输出被 shell profile 污染、MCP server 连接失败、认证未完成，还是 tool search 让 schema 延迟加载？
- 文档没有覆盖：保留配置 diff、hook 输入输出样例、debug log、`/context` 对比、`/mcp` 截图或最小复现仓库，后续 Claude Code 版本升级后再复测。

这篇文章的核心结论可以压成一句话：

```text
Hooks 让控制逻辑在确定节点发生，
MCP 让 Claude Code 连接外部世界；
真正可落地的工程化，是用 Skill 教会用法，用 Hook 固化边界，用 MCP 提供能力。
```

---

## 参考资料

### 官方主参考

- [Automate workflows with hooks](https://code.claude.com/docs/en/hooks-guide)
- [Hooks reference](https://code.claude.com/docs/en/hooks)
- [Connect Claude Code to tools via MCP](https://code.claude.com/docs/en/mcp)
- [Extend Claude Code](https://code.claude.com/docs/en/features-overview)
- [Configure permissions](https://code.claude.com/docs/en/permissions)

### GitHub / 社区补充

- [anthropics/claude-plugins-official](https://github.com/anthropics/claude-plugins-official)
- [subinium/awesome-claude-code](https://github.com/subinium/awesome-claude-code)

GitHub 和社区仓库更适合用来观察别人如何组织 plugin、skill、hook、MCP 示例和实践清单，不用于替代官方文档对 Claude Code 行为的定义。凡是社区经验与官方文档冲突，以官方文档和本地实践观察为准。
