# MCP 与完整 Harness：把外部能力接进同一个工具池

> 社区项目核查日期：2026-06-10
> 官方文档核查日期：2026-07-07
> 作者：wt
> 适合读者：个人开发者、团队技术负责人、想理解 Agent Harness 外部能力接入与完整运行时组合的工程师
> 本文定位：`learn-claude-code` 学习系列第 8 篇，也是本系列收束篇。本文承接第 7 篇多 Agent 协作，解释 MCP 工具如何进入同一个工具池，以及完整 Agent Harness 为什么不是一个“大循环”，而是一组边界清晰的运行时组件。
> 实践建议：本文涉及 MCP server、外部 API、认证、部署类工具和完整 harness 实验。实验时优先使用教学版 mock server 或只读 server；连接真实外部系统前，先确认权限、输出上限、认证方式和破坏性操作审批。
> 配套实验：[AHE-015、AHE-016](../../labs/agent-harness-engineering/00-Agent-Harness-实验手册.md)。

---

## 先把边界说清楚

本文基于社区教学项目 `shareAI-lab/learn-claude-code`，不代表 Anthropic Claude Code 官方内部实现。

本文讨论的是 Claude Code-like Agent Harness 的通用 MCP 接入和完整运行时组合机制，不是在还原 Claude Code 源码。文中使用四类标签：

| 标签 | 含义 |
| --- | --- |
| 【官方事实】 | 来自 Claude Code 官方文档或官方公开材料 |
| 【社区教学实现】 | 来自 `shareAI-lab/learn-claude-code` 的 README 和 `code.py` |
| 【作者推导】 | 基于教学代码、官方边界和工程经验形成的解释 |
| 【动手观察】 | 读者可以本地复现的实验现象 |

特别说明：`learn-claude-code` 是社区教学项目。本文只把它当作理解 Agent Harness 工程机制的可运行样本，不把它写成 Claude Code 官方内部实现。涉及官方能力时，只以 Claude Code 官方文档为准。

前七篇已经逐层搭起了 harness：

```text
Agent Loop
  -> Tool Use
  -> Permission
  -> Hooks
  -> Context Engineering
  -> Recovery / Background / Cron
  -> Multi-Agent / Worktree
```

第 8 篇解决最后一个问题：

> 内置工具已经很多了，外部系统还能不能以同一种方式进入 agent loop？

答案是：可以，但前提是外部能力必须被协议化、命名化、权限化、预算化。

---

## 一、核心认知：MCP 不是“再加几个工具”，而是外部能力的边界层

如果只看表面，MCP 很像：

```text
给模型多几个工具。
```

但这会低估 MCP 在 Agent Harness 里的位置。

真正的问题不是“能不能调外部 API”，而是：

| 问题 | 如果没有边界层会怎样 |
| --- | --- |
| 工具发现 | 每接一个外部系统都要手写 schema 和 handler |
| 命名冲突 | 多个系统都有 `search`、`get`、`update` |
| 权限控制 | 外部 deploy、delete、send_email 等动作可能被误触发 |
| 认证配置 | token、OAuth、动态 header 混在主循环里 |
| 上下文预算 | 几百个外部工具 schema 直接塞爆 prompt |
| 输出体积 | 数据库查询、日志、issue 列表可能淹没上下文 |
| 生命周期 | 外部 server 断线、更新工具、推送消息，都要有恢复路径 |

【作者推导】MCP 的工程价值是把外部系统变成一类标准化能力源：

```text
External System
  -> MCP Server
  -> Tool / Resource / Prompt / Channel
  -> Harness Tool Pool
  -> Agent Loop
```

这和前几篇的线索是统一的：

| 前文机制 | MCP 接入后的对应问题 |
| --- | --- |
| Tool System | MCP tool 也要进入同一套 tool_use / tool_result 协议 |
| Permission | 外部破坏性动作也要被检查 |
| Hooks | 外部工具前后仍要日志、审计、后处理 |
| Context Engineering | MCP 工具太多时要按需发现 |
| Long Task | 外部 server 可能断线、超时、推送异步消息 |
| Multi-Agent | 外部能力应避免随意暴露给所有 teammate |

所以 MCP 不是“功能加法”，而是外部能力进入 harness 的标准入口。

---

## 二、官方事实：Claude Code 公开文档里的 MCP 边界

先看官方公开边界，再看社区教学实现。

【官方事实】Claude Code MCP 文档说明，Claude Code 可以通过 Model Context Protocol 连接外部工具、数据库和 API。MCP server 能让 Claude 读取和操作外部系统，例如 issue tracker、监控系统、数据库、设计工具、邮件等；文档也提醒，连接会获取外部内容的 server 时要确认信任，因为可能带来 prompt injection 风险。

【官方事实】同一文档说明，Claude Code 支持多种 MCP transport：远程 HTTP、远程 SSE、本地 stdio、远程 WebSocket。其中 HTTP 是连接云端服务的推荐方式；SSE 已被标为 deprecated；stdio 适合本地进程和需要本机访问的自定义脚本；WebSocket 适合需要保持双向连接并推送事件的远程 server。

【官方事实】MCP server 可以按 local、project、user 等 scope 配置。local scope 默认只对当前项目和当前用户可见；project scope 会写入项目根目录 `.mcp.json`，适合团队共享；user scope 对用户所有项目可用。官方文档还说明 project-scoped server 可能处于 pending approval 状态，需要在交互会话里审核。

【官方事实】Claude Code 支持 MCP 动态能力更新：server 可以发送 `list_changed` 通知，让 Claude Code 刷新可用工具、prompts 和 resources。HTTP 或 SSE server 断线时，Claude Code 会自动重连并使用指数退避；stdio server 是本地进程，不会自动重连。

【官方事实】Claude Code MCP 文档说明，MCP server 可以作为 channel 把外部事件推入 session；MCP server 也可以提供 resources，用户可通过 `@server:protocol://resource/path` 引用；当 server 支持 resources 时，Claude Code 会提供列出和读取资源的工具。

【官方事实】Claude Code 默认启用 MCP Tool Search。Tool Search 会延迟加载 MCP 工具定义：启动时只加载工具名和 server instructions，Claude 需要时再搜索相关工具，只有实际使用的工具进入上下文。文档也说明没有固定的 per-server tool cap，实际限制来自上下文窗口预算；工具描述和 server instructions 会被截断，因此应写得简洁清楚。

【官方事实】MCP 工具输出有上限和警告机制。官方文档说明，当 MCP 工具输出超过 10,000 tokens 时会显示警告；可以用 `MAX_MCP_OUTPUT_TOKENS` 调整，也建议 server 作者分页或声明最大结果尺寸。MCP server 还可以通过 elicitation 在任务中请求结构化用户输入。

【官方事实】Claude Code Plugins 文档说明，plugin 可以打包 skills、agents、hooks、MCP servers、LSP servers、background monitors 和默认 settings。MCP 文档也说明，plugin-provided MCP servers 会在 plugin 启用后自动启动，工具会和手动配置的 MCP 工具一起出现。

这些官方事实给第 8 篇一个边界：

| 主题 | 官方公开事实能确认什么 | 本文不会声称什么 |
| --- | --- | --- |
| MCP | Claude Code 可通过 MCP 接入外部工具、数据源和 API | 不把 s19 的 mock `MCPClient` 写成官方实现 |
| Transport | HTTP、stdio、WebSocket 等 transport 有不同适用场景 | 不假设所有 server 都用同一种连接方式 |
| Tool Search | MCP 工具默认按需发现，降低 context 占用 | 不把教学版全量组装工具池写成官方当前策略 |
| Resources / Channels | MCP 不只提供 tools，也可提供 resources 和推送事件 | 不把 s19 仅模拟 tools 当成 MCP 全貌 |
| Plugins | plugin 可打包 MCP servers 和其他组件 | 不把 plugin 简化成“插件就是 MCP” |
| 安全 | 外部内容有 prompt injection 风险，认证和输出要受控 | 不默认外部 server 都可信 |

---

## 三、s19：MCP 接入的最小模型是“发现、命名、合并、调用”

【社区教学实现】s19 用一个 mock `MCPClient` 模拟 MCP server：

```python
class MCPClient:
    def register(self, tool_defs, handlers):
        self.tools = tool_defs
        self._handlers = handlers

    def call_tool(self, tool_name, args):
        return self._handlers[tool_name](**args)
```

教学版没有实现真实网络协议、OAuth、resources 或 channels，而是聚焦最小工具链路：

```text
connect_mcp(name)
  -> server discovers tools
  -> mcp_clients[name] = client
  -> assemble_tool_pool()
  -> model sees mcp__server__tool
  -> handler calls MCPClient.call_tool()
```

这个链路里有四个关键设计点。

### 1. 连接不是调用，连接先做能力发现

s19 的 `connect_mcp("docs")` 不会直接搜索文档，而是把 docs server 接入，并发现它有哪些工具：

```text
docs.search
docs.get_version
```

【作者推导】这一步非常重要。外部系统接进 harness 时，第一步不是“执行某个动作”，而是“把能力目录注册进运行时”。

```text
server connected
tool schemas discovered
tool names normalized
tool handlers registered
```

只有这样，后续模型才能用统一的 tool_use 协议调用它。

### 2. 名称必须规范化并带 namespace

s19 把 MCP 工具命名成：

```text
mcp__{server}__{tool}
```

例如：

```text
mcp__docs__search
mcp__deploy__status
mcp__deploy__trigger
```

并用 `normalize_mcp_name()` 把非法字符替换掉。

【作者推导】这解决两个问题：

| 问题 | namespace 的作用 |
| --- | --- |
| 工具名冲突 | 不同 server 的 `search` 不会互相覆盖 |
| 路由明确 | handler 能从工具名反推出 server 和原始 tool |
| 权限可匹配 | 可以按 `mcp__deploy__*` 做权限策略 |
| 日志可读 | 审计日志能看到外部系统来源 |

没有 namespace，MCP 工具池很快会变成一堆难以区分的 `search`、`get`、`update`。

### 3. MCP 工具最终进入同一个 tool pool

s19 的核心函数是 `assemble_tool_pool()`：

```python
def assemble_tool_pool():
    tools = list(BUILTIN_TOOLS)
    handlers = dict(BUILTIN_HANDLERS)
    for server_name, client in mcp_clients.items():
        for tool_def in client.tools:
            tools.append(schema_with_prefixed_name)
            handlers[prefixed] = lambda **kw: client.call_tool(...)
    return tools, handlers
```

【作者推导】这个函数表达了 MCP 接入的本质：

```text
对模型来说：MCP tool 也是 tool schema
对 harness 来说：MCP tool 也是 handler
对 loop 来说：MCP tool 也返回 tool_result
```

这才叫接进同一个工具池。

如果 MCP 工具走另一套分支：

```text
if mcp_tool:
    special_case()
else:
    normal_tool()
```

主循环会越来越复杂，权限、日志、输出裁剪、后台执行也容易漏掉。

### 4. 工具池变了，系统上下文也要变

s19 里，当模型调用 `connect_mcp` 后，会重新组装工具池和 system prompt：

```python
if connect_mcp_was_called:
    tools, handlers = assemble_tool_pool()
    context = update_context(...)
    system = assemble_system_prompt(context)
```

【作者推导】这是动态能力接入的关键点。

Agent 的可用能力不是启动时一次性确定的。连接 MCP server 后，真实状态已经改变：

```text
before: builtin tools only
after: builtin tools + mcp__docs__search + ...
```

下一轮模型调用必须看到新的工具 schema 和新的上下文描述，否则它无法使用刚接入的能力。

---

## 四、MCP 接入后，权限系统不能缺席

s19 的 mock deploy server 提供两个工具：

```text
status: readOnly
trigger: destructive
```

教学版在描述里标注了 readOnly / destructive，但没有实现完整权限系统。s20 在 `permission_hook()` 里对看起来有破坏性的 MCP 工具做提示或拦截。

【作者推导】MCP 最大的风险在这里：

```text
外部工具往往比本地 read_file 更接近真实世界。
```

例如：

| MCP 能力 | 风险 |
| --- | --- |
| GitHub PR merge | 改变远程代码状态 |
| Jira issue update | 改变团队协作状态 |
| database write | 改变生产数据 |
| deploy trigger | 影响线上服务 |
| email send | 对外发送信息 |
| cloud resource delete | 造成成本或可用性事故 |

所以 MCP 工具进入同一个工具池后，也必须进入同一套控制面：

```text
PreToolUse hook
  -> permission policy
  -> audit log
  -> handler
  -> PostToolUse hook
  -> output budget
```

【作者推导】一个稳妥的 MCP 权限模型至少要区分：

| 类型 | 策略 |
| --- | --- |
| read-only 查询 | 默认允许或低风险确认 |
| workspace 写入 | 走本地文件权限和路径检查 |
| 外部状态修改 | 明确确认，记录审计 |
| 生产环境动作 | 默认拒绝或需要更强审批 |
| 大输出查询 | 要分页、limit、摘要或输出上限 |

这也是为什么“工具描述里写 readOnly”不够。描述帮助模型理解，权限层负责强制执行。

---

## 五、Tool Search：MCP 工具多了以后，不能全塞上下文

s19 的教学版为了简单，每次 `assemble_tool_pool()` 都把已连接 server 的所有 tool schema 放入工具列表。

这在小样本里没问题：

```text
docs: search, get_version
deploy: trigger, status
```

但真实 MCP 生态里，一个 server 可能有几十个工具，多个 server 加起来可能有上百个工具。

【官方事实】Claude Code 官方文档说明，MCP Tool Search 默认启用。它会延迟加载 MCP 工具定义：启动时只加载工具名和 server instructions，Claude 需要时再搜索相关工具，只有实际使用的工具进入上下文。

【作者推导】这和第 5 篇 Skill 的渐进式披露是同一个思想：

| 机制 | 常驻上下文 | 按需加载 |
| --- | --- | --- |
| Skill | 名称 + description | 完整 SKILL.md |
| MCP Tool Search | server instructions + 工具索引 | 具体 tool schema |
| Resources | resource 引用 | resource 内容 |

MCP 工具池的规模问题，本质是上下文工程问题。

如果把所有外部工具 schema 全量塞进 prompt，会出现：

| 问题 | 后果 |
| --- | --- |
| token 成本上升 | 每轮都为无关工具付费 |
| 工具选择噪声 | 模型在一堆相似工具里误选 |
| prompt cache 失效 | 动态工具池变化导致缓存不稳定 |
| 描述被截断 | 关键约束可能丢失 |

所以 s19 的教学模型适合理解机制，但生产级 harness 需要 tool search、分层 catalog 或按任务加载。

---

## 六、Resources、Channels、Elicitation：MCP 不只是工具调用

s19 只模拟了 tools，这是刻意简化。

【官方事实】Claude Code MCP 文档说明，MCP server 还可以提供 resources，用户能用 `@server:protocol://resource/path` 引用；server 支持 resources 时，Claude Code 会提供列出和读取资源的能力。

【官方事实】MCP server 也可以通过 channel 把消息推入 session，让 Claude 对 CI、监控告警、聊天消息、webhook 等外部事件作出反应。

【官方事实】MCP server 可以通过 elicitation 在任务中请求结构化输入，例如表单字段或浏览器认证/审批 URL。

【作者推导】这说明 MCP 在 harness 里至少有三类入口：

| MCP 能力 | 进入 harness 的方式 | 类似前文章节 |
| --- | --- | --- |
| Tool | model 主动 tool_use | 第 2 篇工具系统 |
| Resource | 用户或模型引用外部材料 | 第 5 篇上下文工程 |
| Channel | 外部事件推送进 session | 第 6 篇 cron/background notification |
| Elicitation | server 请求用户补充输入 | 第 3 篇权限/确认边界 |

所以 MCP 不应该被理解成“HTTP function calling”。它是外部系统与 agent runtime 的协议边界。

---

## 七、Plugins：把 MCP 和其他扩展打包成可分发能力

MCP 解决的是外部系统接入。

Plugin 解决的是扩展能力如何组织、分发和复用。

【官方事实】Claude Code Plugins 文档说明，plugin 可以包含 skills、agents、hooks、MCP servers、LSP servers、background monitors、默认 settings 等组件。插件适合共享给团队、跨项目复用、版本化发布和 marketplace 分发；单项目快速实验则可以先用 `.claude/` standalone 配置。

【官方事实】MCP 文档说明，plugin-provided MCP servers 会在 plugin 启用后自动连接；它们的工具会和手动配置的 MCP 工具一起出现。plugin 可以通过 `.mcp.json` 或 `plugin.json` 定义 MCP servers。

【作者推导】Plugin 把前面几篇机制打包成一个工程交付单元：

| Plugin 组件 | 对应本系列机制 |
| --- | --- |
| `skills/` | 上下文工程、渐进式披露 |
| `agents/` | subagent / team worker |
| `hooks/` | 生命周期控制面 |
| `.mcp.json` | 外部工具协议接入 |
| `monitors/` | 外部事件 notification |
| `settings.json` | 团队默认行为和权限倾向 |
| `bin/` | 本地可执行工具 |

这也是第 8 篇的一个收束点：

```text
单个工具解决一个动作；
MCP server 接入一组外部能力；
Plugin 打包一套可复用工作方式；
完整 harness 负责把它们都纳入同一个运行时。
```

---

## 八、s20：完整 Harness 不是大函数，而是统一协议面

s20 的标题是 Comprehensive Agent。很容易误解成：

```text
把前面所有代码粘成一个超大 code.py。
```

但真正要看的不是文件长度，而是组件在 loop 中的位置。

【社区教学实现】s20 的 README 把组件位置整理成一张表：

| 位置 | 组件 |
| --- | --- |
| 用户输入前后 | `UserPromptSubmit` hooks |
| LLM 前 | cron queue、background notifications、compaction pipeline、memory、skills、MCP state |
| LLM 调用 | error recovery |
| 工具执行前 | `PreToolUse` hooks + permission |
| 工具分发 | `assemble_tool_pool()` |
| 工具执行时 | background dispatch |
| 工具执行后 | `PostToolUse` hooks |
| 返回循环 | tool_result |
| 停止时 | `Stop` hooks |

【作者推导】这张表是本系列最重要的收束。

完整 harness 的核心不是“功能多”，而是每类功能都有固定插入点：

```text
外部事件先注入
上下文先整理
工具池先组装
模型再决策
工具前先检查
工具后再回填
状态变化再更新
下一轮重新开始
```

s20 的 `agent_loop()` 可以压缩成：

```python
while True:
    inject_cron_jobs()
    inject_background_notifications()
    prepare_context(messages)
    context = update_context(context, messages)
    tools, handlers = assemble_tool_pool()

    response = call_llm(messages, context, tools, recovery_state)

    for tool_use in response.tool_uses:
        if pre_tool_hook_blocks(tool_use):
            append_blocked_result()
        elif should_run_background(tool_use):
            start_background_task()
        else:
            output = call_tool_handler(handler, tool_use.input)
            post_tool_hooks(tool_use, output)
            append_tool_result(output)
```

这段伪代码说明了一件事：

> 所有机制最终都要回到同一个 tool_use / tool_result / messages 循环。

如果某个机制不能进入这个循环，它就会变成旁路状态，模型看不见、权限管不到、日志审不到、压缩也处理不了。

---

## 九、完整 Harness 的组件分层

经过八篇文章，可以把 Claude Code-like Agent Harness 抽象成七层：

| 层 | 负责什么 | 典型组件 |
| --- | --- | --- |
| Model Protocol | 和模型交互 | messages、system、tools、tool_use、tool_result |
| Tool Runtime | 执行动作 | builtin handlers、MCP handlers、background dispatch |
| Control Plane | 确定性控制 | permissions、hooks、audit、output limits |
| Context Plane | 上下文预算 | todo、skills、memory、compact、prompt assembly |
| Persistence Plane | 长任务状态 | task files、transcripts、cron jobs、worktrees |
| Collaboration Plane | 多 Agent 协作 | subagent、teammate、mailbox、protocol |
| Extension Plane | 外部能力 | MCP、plugins、resources、channels、monitors |

【作者推导】这七层的边界比具体代码更重要。

常见错误是把所有东西都塞进 prompt 或主 loop：

```text
让 prompt 记住权限
让 loop 手写每个外部 API
让 memory 保存当前任务状态
让 tools 自己决定是否安全
让 teammate 直接改主工作区
```

更稳的做法是：

```text
prompt 负责解释目标和可用能力
permission 负责强制边界
tool runtime 负责执行
context plane 负责预算
persistence plane 负责恢复
collaboration plane 负责多人状态
extension plane 负责外部能力接入
```

这就是本系列反复强调的结论：

```text
Agent 不是一个 prompt。
Agent 是一个 harness。
```

---

## 十、从 s01 到 s20 的主线回看

`learn-claude-code` 的 20 个章节可以看成一条逐步扩展的工程路径。

| 阶段 | 章节 | 机制主线 |
| --- | --- | --- |
| 最小行动 | s01-s02 | 模型通过 tool_use / tool_result 形成行动循环 |
| 安全控制 | s03-s04 | 权限和 hooks 把确定性控制放到 loop 外围 |
| 上下文管理 | s05、s07-s10 | Todo、Skill、Compact、Memory、System Prompt 管理不同生命周期的信息 |
| 稳定运行 | s11-s14 | 错误恢复、任务系统、后台任务、Cron 支撑长任务 |
| 协作扩展 | s06、s15-s18 | Subagent、Team、Protocol、Autonomous、Worktree 支撑多 Agent |
| 外部能力 | s19 | MCP 把外部系统接进工具池 |
| 综合运行时 | s20 | 所有机制回到同一个 agent loop |

【作者推导】这条线也解释了学习 Agent Harness 的正确顺序：

```text
先理解 loop
再理解 tool
再理解 permission / hooks
再理解 context
再理解 recovery
再理解 collaboration
最后理解 extension
```

反过来，如果一开始就看 MCP、plugins、多 Agent，很容易只看到“功能很多”，看不到它们为什么必须被放进同一套运行时协议。

---

## 十一、几个容易踩的坑

### 1. 把 MCP 当成普通 HTTP wrapper

普通 HTTP wrapper 只回答：

```text
怎么调用这个 API？
```

MCP 接入要回答：

```text
怎么发现能力？
怎么描述 schema？
怎么命名避免冲突？
怎么认证？
怎么限制输出？
怎么处理断线？
怎么让权限层识别破坏性操作？
怎么让模型按需找到工具？
```

所以 MCP server 的质量不只取决于 API 能不能通，还取决于它给 harness 的描述、分页、权限语义和错误语义是否清楚。

### 2. 把所有 MCP 工具全量塞进上下文

教学版 s19 这么做是为了简单。真实工具生态里，这会带来明显成本。

更好的方向是：

```text
server instructions 常驻
tool catalog 可搜索
具体 schema 按需加载
结果分页或摘要
```

这也是官方 Tool Search 默认启用的原因。

### 3. 只看工具，不看 resources 和 channels

很多外部系统并不只适合“调用函数”。

| 外部能力 | 更合适的 MCP 形态 |
| --- | --- |
| issue 详情 | resource |
| 数据库 schema | resource |
| CI 失败通知 | channel |
| 登录或审批 | elicitation |
| 触发部署 | tool + permission |

只用 tools 视角看 MCP，会漏掉很多运行时入口。

### 4. MCP 工具绕过权限系统

外部工具越强，越不能绕过权限。

【作者推导】破坏性 MCP 工具应该至少满足：

```text
工具描述明确 destructive
schema 要求必要参数
PreToolUse hook 可识别
高风险动作需要确认
执行结果进入审计日志
失败结果不被包装成成功
```

### 5. 完整 harness 变成大泥球

s20 的风险是：看起来所有机制都在一个文件里。

学习时可以这样，因为它帮助看到全链路。真正工程化时，应该按平面拆开：

```text
tool registry
permission engine
hook dispatcher
context manager
task store
background runner
scheduler
mcp connector
agent loop
```

拆开的目标不是抽象漂亮，而是让每个机制有清楚职责和测试边界。

---

## 十二、最小代码理解

下面只列最小代码点，目的是理解机制，不是复刻完整实现。

| 章节 | 最小代码点 | 机制含义 |
| --- | --- | --- |
| s19 | `MCPClient.register()` | server 暴露工具 schema 和 handler |
| s19 | `connect_mcp(name)` | 连接 server 并发现工具 |
| s19 | `normalize_mcp_name()` | 防止外部名称污染工具命名空间 |
| s19 | `mcp__server__tool` | 用 namespace 避免冲突并方便路由 |
| s19 | `assemble_tool_pool()` | 把 builtin tools 和 MCP tools 合并到同一工具池 |
| s19 | `handlers[prefixed] = ...call_tool(...)` | MCP 工具最终仍是 handler |
| s19 | connect 后重建 tools/system | 动态能力变更后下一轮模型要看到 |
| s20 | `prepare_context()` | 每轮 LLM 前统一做上下文预算处理 |
| s20 | `call_llm(..., RecoveryState)` | 模型调用统一经过恢复层 |
| s20 | `trigger_hooks("PreToolUse", block)` | 工具执行前统一走控制面 |
| s20 | `should_run_background()` | 慢工具异步化，不阻塞 loop |
| s20 | `build_user_content(results)` | 工具结果和后台通知都回到 user-side content |
| s20 | `consume_cron_queue()` | 外部时间事件回到同一 loop |
| s20 | `update_context()` | 根据真实状态重新组装上下文 |

可以把完整 harness 压缩成一个不依赖具体产品的伪代码：

```python
while session_active:
    messages += external_events.poll()
    messages = context_manager.prepare(messages)
    context = state_reader.snapshot()
    tools, handlers = tool_registry.assemble(context)

    response = model.call(system=context.system, messages=messages, tools=tools)

    for tool_use in response.tool_uses:
        decision = control_plane.before_tool(tool_use)
        if decision.blocked:
            results.append(blocked_result(tool_use, decision.reason))
            continue

        if runtime.should_background(tool_use):
            results.append(runtime.start_background(tool_use))
            continue

        output = handlers[tool_use.name](**tool_use.input)
        control_plane.after_tool(tool_use, output)
        results.append(tool_result(tool_use, output))

    messages.append({"role": "user", "content": results})
```

这段伪代码就是本系列的核心：

```text
模型只决定 tool_use；
harness 负责权限、执行、状态、恢复、回填和扩展。
```

一个 mock MCP server 的最小运行记录可以这样整理：

| 步骤 | 输入 | 可观察输出 | 机制结论 |
| --- | --- | --- | --- |
| 连接 server | `connect_mcp("docs")` | 返回 `docs.search` 等工具定义 | 连接阶段先发现能力，不直接执行业务 |
| 规范化命名 | `docs.search` | `mcp__docs__search` | namespace 让路由和权限可控 |
| 合并工具池 | builtin + MCP tools | 下一轮 `tools` 增加 MCP schema | 外部能力进入同一 tool pool |
| 调用工具 | `mcp__docs__search(query)` | MCP handler 返回搜索结果 | 外部工具仍走 handler |
| 回填结果 | handler output | `tool_result` 进入 messages | 模型只能基于回填后的 observation 继续 |

这个记录比“连上 MCP 了”更有价值，因为它能证明 MCP 没有绕开 harness 的统一协议面。

---

## 十三、动手实验

以下实验建议在临时目录中完成。如果你本地没有 `learn-claude-code`，先 clone 到临时目录；不要把实验输出写进本文所在写作仓库。

本节实验对应实验手册编号：AHE-015、AHE-016。

### 实验 1：观察 s19 的 MCP 工具发现

输入：

```text
Connect to the docs MCP server, then search docs for "authentication".
```

预期观察：

| 现象 | 说明 |
| --- | --- |
| `connect_mcp("docs")` 返回 discovered tools | 连接阶段完成能力发现 |
| 下一轮工具池出现 `mcp__docs__search` | MCP 工具被合并进 tool pool |
| 搜索结果以 tool_result 回填 | MCP 工具和内置工具走同一 feedback loop |

验证标准：

```text
connect_mcp 本身不是搜索；
连接后重建工具池，模型才能调用新工具。
```

### 实验 2：观察 MCP 工具命名隔离

输入：

```text
Connect to docs and deploy MCP servers.
List available tools or use docs search and deploy status.
```

预期观察：

| 现象 | 说明 |
| --- | --- |
| 出现 `mcp__docs__search` | docs server 的 search 被 namespace 包住 |
| 出现 `mcp__deploy__status` | deploy server 的 status 独立命名 |
| 没有裸 `search` 覆盖内置工具 | namespace 防冲突 |

验证标准：

```text
多个 server 即使有同名工具，也必须能稳定路由。
```

### 实验 3：观察 destructive MCP 工具的权限边界

输入：

```text
Connect to deploy MCP server.
Try to trigger deployment for "demo-service".
```

预期观察：

| 现象 | 说明 |
| --- | --- |
| deploy trigger 被识别为高风险或 destructive | 外部状态修改不能等同 read-only |
| permission/hook 有机会介入 | MCP 工具也应进入控制面 |
| 结果以 tool_result 返回 | 即使被拒绝，也要让模型看到拒绝原因 |

验证标准：

```text
MCP 工具不能绕过 PreToolUse；
破坏性动作不能只靠模型自觉避免。
```

### 实验 4：跑 s20 看完整 loop 顺序

输入：

```text
Create a todo, connect docs MCP, schedule a reminder, then run a slow command in background.
```

预期观察：

| 现象 | 说明 |
| --- | --- |
| todo reminder、MCP state、cron、background 都进入同一 loop | 完整 harness 不是多条旁路 |
| 慢命令先返回占位结果 | background dispatch 生效 |
| 后台完成后 notification 注入 | 异步结果回流 |
| MCP 工具仍走 tool_result | 外部能力进入统一协议 |

验证标准：

```text
所有状态变化最终都回到 messages、context、tools、handlers 这组核心结构。
```

### 实验 5：给一个 mock MCP server 加输出限制

输入：

```text
Modify the docs mock server so search returns a very long string.
Then observe whether large_output_hook or compact pipeline catches it.
```

预期观察：

| 现象 | 说明 |
| --- | --- |
| 大输出被警告或压缩 | 外部工具输出不能无限进入上下文 |
| 后续 compact pipeline 介入 | MCP 输出也属于上下文预算 |
| 模型仍能继续工作 | 输出治理不应破坏 loop |

验证标准：

```text
外部工具接入后，输出大小和可恢复性仍受 harness 管理。
```

---

## 十四、自测问题

1. MCP 和普通 HTTP wrapper 的区别是什么？
2. 为什么 MCP 工具需要 `mcp__server__tool` 这样的 namespace？
3. `connect_mcp` 为什么要触发工具池和 system prompt 重建？
4. Tool Search 解决的是工具调用问题，还是上下文预算问题？
5. MCP resources、channels、tools 分别对应 harness 里的哪类输入？
6. 为什么 plugin 不是 MCP 的同义词？
7. s20 的完整 harness 为什么不应该被理解为“一个巨大循环”？
8. 如果一个 MCP server 暴露 100 个工具，你会如何控制上下文、权限和输出？

---

## 参考资料

官方文档：

- [Claude Code Docs: Connect Claude Code to tools via MCP](https://code.claude.com/docs/en/mcp)
- [Claude Code Docs: Create plugins](https://code.claude.com/docs/en/plugins)
- [Claude Code Docs: Claude Code settings](https://code.claude.com/docs/en/settings)
- [Claude Code Docs: Hooks reference](https://code.claude.com/docs/en/hooks)
- [Claude Code Docs: Automate actions with hooks](https://code.claude.com/docs/en/hooks-guide)
- [Claude Code Docs: Work with sessions](https://code.claude.com/docs/en/agent-sdk/sessions)

社区教学项目：

- [shareAI-lab/learn-claude-code: s19_mcp_plugin](https://github.com/shareAI-lab/learn-claude-code/tree/main/s19_mcp_plugin)
- [shareAI-lab/learn-claude-code: s20_comprehensive](https://github.com/shareAI-lab/learn-claude-code/tree/main/s20_comprehensive)
