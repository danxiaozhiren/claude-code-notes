# Agent Harness 工程学习系列收尾审计报告

> 审计日期：2026-06-29
> 审计对象：`tracks/agent-harness-engineering/`、`labs/agent-harness-engineering/`、`references/agent-harness-glossary.md`
> 来源边界：本报告只审计本仓库写作材料；`shareAI-lab/learn-claude-code` 仍然是远程社区教学参考，不代表 Anthropic Claude Code 官方内部实现。

---

## 一、总判断

当前仓库已经从“单组文章”进入“可长期扩展的学习库”状态。`tracks/` 放系列文章，`labs/` 放可复现实验，`references/` 放统一术语，`meta/` 放规划和审计材料，这个结构后续可以继续承载新的学习方向。

Agent Harness 8 篇文章整体不是浅尝辄止。它们已经覆盖 Agent Loop、Tool Runtime、Control Plane、Context Plane、稳定性、多 Agent 协作、MCP 和完整 harness 组合，并且大多数文章都有“作者推导”“最小代码理解”“动手实验”。

当前最需要补强的不是继续新增第 9 篇，而是让已有材料形成闭环：

| 闭环 | 当前状态 | 本轮处理 |
| --- | --- | --- |
| 入口闭环 | 顶层 README、tracks、labs、references 已存在 | 补 `meta/agent-harness-engineering/` 入口 |
| 文章闭环 | 8 篇文章已完成 | 补每篇文章对应 AHE 实验编号 |
| 实验闭环 | 16 个实验覆盖 8 篇文章 | 补风险分级和高风险实验边界 |
| 术语闭环 | 已有 Agent Harness 术语表 | 后续逐篇校对术语一致性 |
| 深度闭环 | 多数文章已达到工程推导层 | 后续优先补少量“为什么这样设计”的显式收束 |

## 二、结构审计

| 区域 | 审计结果 | 建议 |
| --- | --- | --- |
| `README.md` | 能说明仓库定位、目录结构、资料边界和新增方向规范 | 后续新增 track 时只更新当前学习方向表 |
| `tracks/README.md` | 能解释 track 扩展方式 | 保持轻量，不放长篇规划 |
| `tracks/agent-harness-engineering/README.md` | 有边界说明、文章列表、阅读路线和实验路线 | 已补文章到 AHE 实验编号映射 |
| `labs/README.md` | 已列出 Claude Code 和 Agent Harness 两组实验 | 后续实验集继续按 `labs/<topic>/` 扩展 |
| `references/README.md` | 已说明术语表和来源类型 | 后续新增资料索引时继续集中到 `references/` |
| `meta/README.md` | 原来只登记官方能力系列规划 | 已补 Agent Harness 审计材料入口 |

结构结论：顶层目录不需要再重排。新增学习方向时继续使用 `tracks/<topic>/`，必要时补 `labs/<topic>/` 和 `references/` 公共材料。

## 三、文章深度审计

深度等级按这四档判断：

| 等级 | 含义 |
| --- | --- |
| 入门说明 | 能解释是什么，但主要停留在概念导读 |
| 机制解释 | 能解释组件如何运行、边界在哪里 |
| 工程推导 | 能解释为什么这样设计、取舍是什么 |
| 可复现实验 | 能用实验观察支撑机制理解 |

| 篇号 | 主题 | 当前等级 | 依据 | 后续补强点 |
| --- | --- | --- | --- | --- |
| 1 | 最小 Agent Loop | 工程推导 | 能从 `messages`、`tool_use`、`tool_result` 推出 harness 骨架 | 可补一张更紧凑的 loop 状态图 |
| 2 | 工具系统 | 可复现实验 | 有 schema / handler / dispatch / failure / MCP 过渡和多组实验 | 可把 MCP 过渡更明确地指向第 8 篇 |
| 3 | 权限系统 | 工程推导 | 能区分权限、提示词建议、hook、MCP 权限边界 | 可补一张 allow / ask / deny 决策表 |
| 4 | Hooks | 工程推导 | 能解释 hook 是生命周期控制点，不是模型主动工具 | 可补 hook 失败时的 fail-closed 策略 |
| 5 | 上下文工程 | 可复现实验 | Todo、Skill、Compact、Memory、System Prompt 的生命周期区分清楚 | 可补一个 compact 前后上下文对照样例 |
| 6 | 失败恢复与长任务 | 工程推导 | 覆盖 retry、continuation、reactive compact、task、background、cron | 可补恢复状态机图 |
| 7 | 多 Agent 协作 | 工程推导 | 能区分 subagent、mailbox、team protocol、worktree 的职责 | 可补 lead / teammate 的消息时序图 |
| 8 | MCP 与完整 Harness | 可复现实验 | MCP discovery、namespace、tool pool、permission、resources、channels、plugin 和 s20 收束完整 | 可补一个 mock MCP server 最小运行记录 |

深度结论：系列主体已经超过“皮毛”层。下一轮不应重写为更长的文章，而应补少量图、表、运行记录，让关键推导更可复读。

## 四、概念一致性审计

| 概念 | 当前使用情况 | 需要保持的写法 |
| --- | --- | --- |
| Agent Harness | 已作为系列核心术语 | 指模型外侧 runtime，不等同于 prompt |
| Agent Loop | 已稳定写成 `model -> tool_use -> handler -> tool_result -> model` | 不写成固定工作流或单轮函数调用 |
| Tool Runtime | 已从 schema、handler、dispatch、tool_result 展开 | 不写成“模型直接执行工具” |
| Control Plane | 权限、hooks、审计已基本归位 | 强调确定性边界在 runtime，不靠模型自觉 |
| Context Plane | Todo、Skill、Compact、Memory、System Prompt 已分层 | 避免把所有机制都叫 memory |
| Subagent / Team | 第 7 篇已区分上下文隔离和团队协作 | 避免把 subagent 简化成完整团队系统 |
| MCP | 第 8 篇已写成外部能力协议层 | 不写成普通 HTTP wrapper |

来源边界结论：8 篇文章开头都声明了社区教学项目边界，并用【官方事实】【社区教学实现】【作者推导】【动手观察】区分信息类型。后续复用或发布前，需要重新核查官方文档日期。

## 五、实验闭环审计

| 文章 | 配套实验 | 覆盖关系 |
| --- | --- | --- |
| 1 | AHE-001、AHE-002 | 最小 loop 和工具结果回填 |
| 2 | AHE-002、AHE-003 | tool_use / tool_result 对齐和工具扩展 |
| 3 | AHE-004、AHE-016 | 权限拒绝回填和完整执行链路 |
| 4 | AHE-005、AHE-016 | hook 生命周期和完整 harness 插入点 |
| 5 | AHE-006、AHE-007、AHE-008 | Todo / Task、Skill、Compact / Memory |
| 6 | AHE-006、AHE-008、AHE-009、AHE-010、AHE-011 | 文件化任务、压缩恢复、后台和定时触发 |
| 7 | AHE-012、AHE-013、AHE-014 | Subagent、Mailbox / Protocol、Worktree |
| 8 | AHE-015、AHE-016 | MCP discovery、namespace、完整 harness |

实验结论：16 个实验可以覆盖 8 篇文章。高风险实验已经补充临时 clone、mock server、禁用真实部署和避免真实项目目录的边界。

## 六、后续修改优先级

| 优先级 | 修改项 | 目标 |
| --- | --- | --- |
| P0 | 保持来源边界和实验安全提示 | 防止把社区教学实现写成官方内部实现 |
| P1 | 给第 1、3、4、6、7 篇补小图或状态表 | 让“为什么这样设计”更容易复读 |
| P1 | 给第 5、8 篇补一个最小运行记录 | 让上下文和 MCP 机制更可观察 |
| P2 | 统一每篇“动手实验”标题和 AHE 编号写法 | 让文章和实验手册可以互相跳转 |
| P2 | 做一次官方链接和日期复核 | 复用或发布前降低时效性风险 |

下一阶段建议：先做 P1 小幅打磨，不新增第三条学习方向。等这组文章达到“可发布版本”后，再考虑开 OpenAI Agents SDK、LangGraph、SWE-agent 或 MCP server 设计的新 track。
