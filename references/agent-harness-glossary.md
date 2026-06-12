# Agent Harness 术语表

> 更新时间：2026-06-11
> 适用范围：`claude-code-notes` 中的 AI Agent / Claude Code-like Harness 学习笔记
> 来源边界：本文是个人学习术语表，基于本仓库文章、Claude Code 官方公开文档和 `shareAI-lab/learn-claude-code` 社区教学项目整理，不代表 Anthropic Claude Code 官方内部术语。

---

## 使用说明

这个术语表用于统一本仓库后续文章中的说法，避免同一个概念在不同文章里反复改名。

术语解释按四类来源边界理解：

| 类型 | 含义 |
| --- | --- |
| 官方事实 | 来自官方公开文档，可用于描述公开能力和公开概念 |
| 社区教学实现 | 来自 `learn-claude-code` 的 README 与 `code.py`，用于理解工程样本 |
| 作者推导 | 基于官方事实、教学代码和工程经验形成的解释 |
| 动手观察 | 可以通过本地实验复现的现象 |

---

## 一、核心运行时

| 术语 | 简短定义 | 解决的问题 | 关联文章 |
| --- | --- | --- | --- |
| Agent Harness | 包在模型外面的运行时，负责编排上下文、工具、权限、状态、恢复和扩展 | 让 LLM 从“生成文本”变成“受控行动” | Agent Harness 1、8 |
| Agent Loop | `model -> tool_use -> handler -> tool_result -> model` 的循环 | 让模型可以连续观察、行动、再观察 | 1 |
| Messages | 模型每轮看到的工作现场，包含用户目标、模型回复、工具结果和系统注入信息 | 保存当前任务的可见状态 | 1、5、8 |
| System Prompt | 每轮模型调用前由 harness 组装的系统输入 | 把身份、工具、规则、记忆、workspace 等状态组织给模型 | 5、8 |
| Tool Use | 模型输出的结构化工具调用请求 | 把“想做什么”表达成可执行意图 | 1、2 |
| Tool Result | harness 执行工具后回填给模型的结果 | 让外部世界的执行结果进入下一轮推理 | 1、2 |
| Handler | Python/JS/CLI 等真实执行函数 | 把 tool_use 变成外部动作 | 2、8 |
| Tool Pool | 当前模型可见的工具 schema 集合和对应 handler 表 | 管理 agent 这一轮能做什么 | 2、8 |
| Dynamic Tool Pool | 随状态变化重新组装的工具池，例如连接 MCP 后新增外部工具 | 让运行时能力可动态扩展 | 2、8 |

## 二、工具与外部能力

| 术语 | 简短定义 | 解决的问题 | 关联文章 |
| --- | --- | --- | --- |
| Builtin Tool | harness 本地内置工具，如读写文件、shell、todo | 提供基础工程动作 | 2、8 |
| MCP | Model Context Protocol，外部系统向 agent 暴露 tools/resources/prompts 等能力的协议 | 标准化接入外部工具、数据库和 API | 2、3、8 |
| MCP Server | 提供 MCP 能力的外部或本地服务 | 把某个系统封装成可发现、可调用的能力源 | 8 |
| MCP Tool | MCP server 暴露给模型调用的工具 | 让外部系统进入同一 tool_use / tool_result 协议 | 8 |
| Tool Namespace | 工具命名空间，例如 `mcp__server__tool` | 避免外部工具重名，方便路由、授权和审计 | 3、8 |
| Tool Search | 按需发现和加载工具 schema，而不是把所有工具全塞上下文 | 降低大工具池的上下文成本和选择噪声 | 5、8 |
| Resource | MCP server 暴露的可引用外部材料 | 让外部文档、issue、schema 等作为上下文材料进入 session | 8 |
| Channel | MCP server 向 session 推送外部事件的通道 | 让 CI、告警、webhook 等异步事件进入 agent loop | 6、8 |
| Elicitation | MCP server 在任务中请求用户补充结构化输入 | 处理登录、审批、表单字段等人机交互 | 3、8 |
| Plugin | 可分发的扩展包，可包含 skills、agents、hooks、MCP servers、settings 等 | 把一套工作方式打包复用 | 5、8 |

## 三、控制面

| 术语 | 简短定义 | 解决的问题 | 关联文章 |
| --- | --- | --- | --- |
| Permission | handler 执行前的强制边界，不是提示词建议 | 防止 agent 自主行动越界 | 3 |
| Permission Mode | 对权限信任级别的配置，例如默认、自动接受编辑、计划模式等 | 在不同环境中切换审批策略 | 3 |
| Allow / Ask / Deny | 权限决策的三类结果 | 明确工具调用是直接执行、询问用户还是拒绝 | 3 |
| Fail Closed | 无法确认安全时默认拒绝或保留 | 避免不确定状态下误执行破坏性动作 | 3、7 |
| Hook | harness 生命周期点上的自动触发逻辑 | 把日志、权限、审计、后处理等横切控制挂到 loop 外 | 4 |
| PreToolUse | 工具执行前触发的 hook 点 | 拦截危险工具调用、做权限判断 | 3、4、8 |
| PostToolUse | 工具执行成功后触发的 hook 点 | 做日志、输出检查、后处理 | 4、8 |
| Stop Hook | 模型不再请求工具、准备结束时触发 | 总结、清理、阻止过早停止 | 4、6 |
| Audit | 对工具调用、权限决策、外部动作的可追踪记录 | 让 agent 行动可复盘 | 3、4、8 |

## 四、上下文面

| 术语 | 简短定义 | 解决的问题 | 关联文章 |
| --- | --- | --- | --- |
| Context Engineering | 按生命周期管理进入模型上下文的信息 | 在有限 context window 内保留关键状态、减少噪声 | 5 |
| Todo | 当前会话内的轻量任务步骤 | 让模型看见当前目标和进度 | 5 |
| Task | 文件化、可依赖、可认领的任务对象 | 支撑长任务、多 agent 协作和恢复 | 6、7 |
| Skill | 按需加载的领域流程、规则和方法知识 | 避免把所有操作规程常驻 system prompt | 5 |
| Memory | 跨会话、跨压缩保留的长期事实和偏好 | 让重要背景不只依赖当前 messages | 5 |
| Compact | 对长对话、工具结果或历史进行压缩 | 避免上下文超限 | 5、6、8 |
| Reactive Compact | prompt too long 之后触发的紧急压缩和重试 | 让超长上下文错误有恢复路径 | 6、8 |
| Prompt Assembly | 每轮根据真实状态组装 system prompt | 保证模型看到当前工具、记忆、workspace 和扩展状态 | 5、8 |
| Tool Result Budget | 对工具输出体积做裁剪、持久化或摘要 | 防止外部输出淹没上下文 | 5、8 |

## 五、稳定性与长任务

| 术语 | 简短定义 | 解决的问题 | 关联文章 |
| --- | --- | --- | --- |
| Recovery State | 记录恢复尝试的状态对象 | 防止无限重试、重复升级或错误吞掉 | 6、8 |
| Retry / Backoff | 临时错误后的指数退避重试 | 处理 429、529、网络抖动等临时失败 | 6 |
| Continuation Prompt | 输出截断后要求模型继续的提示 | 处理 `max_tokens` 截断而不是从头重做 | 6 |
| Background Task | 慢操作放到后台执行，先返回占位结果 | 避免 build/test/install 阻塞主 loop | 6、8 |
| Task Notification | 后台任务完成后注入给模型的通知 | 让异步结果回到可见上下文 | 6、8 |
| Cron Scheduler | 按时间规则生产工作事件的调度器 | 让未来时间也能重新触发 agent 工作 | 6 |
| Scheduled Prompt | cron 命中后注入 messages 的 prompt | 把定时事件转成同一 agent loop 的输入 | 6、8 |
| Transcript | 会话历史或压缩前上下文的持久化记录 | 支撑复盘、恢复和调试 | 5、6 |

## 六、协作面

| 术语 | 简短定义 | 解决的问题 | 关联文章 |
| --- | --- | --- | --- |
| Subagent | 拿干净上下文处理子任务并返回 summary 的 worker | 隔离子任务噪声，避免污染主会话 | 1、7 |
| Teammate | 可持续运行、可收发消息、可认领任务的队友 agent | 支撑长期并行协作 | 7 |
| Lead | 协调任务、生成队友、汇总结果的主 agent | 管理团队目标和结果综合 | 7 |
| MessageBus | agent 间的消息投递层 | 让队友结果、请求、指令异步回流 | 7 |
| Mailbox | 每个 agent 独立收件箱 | 避免多 agent 消息混杂 | 7 |
| Team Protocol | 带类型、request_id 和状态的协作协议 | 让消息从自然语言聊天变成可路由事件 | 7 |
| ProtocolState | 记录请求类型、发送方、目标、状态和 payload 的对象 | 跟踪审批、停机等跨 agent 请求 | 7 |
| Request ID | 用于匹配请求和响应的唯一 ID | 避免多条审批或停机消息混淆 | 7 |
| Owner | 任务当前负责人 | 防止多个 agent 重复认领同一任务 | 6、7 |
| blockedBy | 当前任务依赖的上游任务 ID 列表 | 防止任务过早启动 | 6、7 |
| Autonomous Agent | 空闲后能检查 inbox 和任务看板、自动认领任务的 agent | 减少 lead 手动分配负担 | 7 |
| Worktree | git 的独立工作目录和分支 | 隔离并行文件修改，避免直接覆盖主工作区 | 7 |

## 七、常见误用

| 误用 | 更准确的理解 |
| --- | --- |
| Agent 就是 prompt | Agent 是模型外加 harness/runtime |
| Tool 就是函数调用 | Tool 是模型协议、schema、handler、权限和回填的组合 |
| Permission 是提醒模型 | Permission 是 runtime 在 handler 前的强制检查 |
| Hook 是工具 | Hook 是生命周期自动触发点，不由模型主动选择 |
| Memory 是长期聊天记录 | Memory 是跨会话可加载的长期事实，不是所有历史 |
| Compact 是随便摘要 | Compact 是上下文预算管理，必须保留可继续工作的状态 |
| Subagent 就是 team | Subagent 主要解决上下文隔离，team 还需要任务、消息和协议 |
| Worktree 解决协作 | Worktree 只解决文件隔离，不解决任务分配和消息协议 |
| MCP 是 HTTP wrapper | MCP 是外部能力进入 harness 的协议层 |
| Plugin 就是 MCP | Plugin 可以打包 MCP，也可以打包 skills、agents、hooks、settings 等 |

## 八、推荐统一用词

| 建议用词 | 尽量避免 |
| --- | --- |
| Agent Harness / Agent Runtime | “智能体 prompt” |
| tool_use / tool_result | “模型直接执行工具” |
| handler 执行工具 | “模型执行命令” |
| 权限层 / 控制面 | “提示词限制” |
| 上下文预算 | “记忆不够” |
| 文件化任务图 | “更高级 Todo” |
| 异步结果回流 | “后台跑完就行” |
| 外部能力接入 | “加几个 API” |

