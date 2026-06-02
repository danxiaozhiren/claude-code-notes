# 多 Agent 架构：Subagents、Agent Teams 与 Worktrees

> 文档核查日期：2026-05-19  
> 作者：wt  
> 写作初衷：在领导的鼓励与倡导下，围绕 Claude Code 做一次面向底层原理的系统学习，把学习过程中的理解、积累和实践经验沉淀下来，并以文章形式分享给更多同事和读者。  
> 适合读者：团队负责人、平台开发者、处理复杂任务的个人开发者  
> 本文定位：Claude Code 系列第 6 篇，承接第 5 篇的扩展能力，解释复杂任务如何拆成多个并行、隔离或协作的执行单元。  
> 实践建议：本文末尾提供动手练习，建议读者用测试仓库观察 subagent context 隔离、worktree 文件隔离和 Agent Teams 协作开销。

---

## 先把边界说清楚

第 5 篇讲了两类扩展能力：Hooks 用来固化控制，MCP 用来连接外部世界。

这篇进入另一个问题：

> 当一个任务太大、太杂、太容易污染 context，Claude Code 如何拆成多个执行单元？

本文关注三件事：

- Subagents：当前 session 内的隔离 worker
- Agent Teams：多个独立 Claude Code sessions 的协作系统
- Worktrees：让并行改动互不覆盖的文件系统隔离

它们经常一起出现，但不是一回事：

```text
Subagent 解决 context 隔离。
Agent Teams 解决多 session 协作。
Worktrees 解决文件改动隔离。
```

不展开的内容：

- Skills、Plugins、Marketplaces：放到第 7 篇
- GitHub Actions、Headless、Agent SDK：放到第 8 篇
- `claude agents` Agent view 和 `/batch`：它们也属于并行工作方式，但本文只在对比中点到，不展开。

本文只在容易混淆的地方标注来源类型：

- 【官方事实】：来自 Claude Code 官方文档或官方仓库。
- 【作者推导】：基于官方事实整理出的机制解释和工程判断。
- 【实践练习】：读者可以在自己的环境中复现观察的部分。

---

## 一、核心认知：并行不是目的，隔离才是关键

很多人看到“多 Agent”，第一反应是：

```text
是不是开得越多越快？
```

这其实是误解。

【官方事实】Claude Code 官方并行工作文档把 subagents、agent view、agent teams、worktrees、batch 放在同一类“agents and parallel work”里，但明确说它们解决的问题不同：subagents 是单 session 内的委托 worker；agent teams 是带共享任务列表和消息系统的多 session 协作；worktrees 是隔离文件改动；不同方式会增加 token 使用。

【作者推导】多 Agent 架构真正要解决的不是“多开几个 Claude”，而是三种隔离：

| 隔离类型 | 解决什么 | 典型能力 |
| --- | --- | --- |
| Context 隔离 | 大量搜索、日志、文件读取不要污染主会话 | Subagents |
| 协作隔离 | 不同 worker 有自己的任务、视角和通信 | Agent Teams |
| 文件系统隔离 | 并行修改不要互相覆盖 | Worktrees |

如果你没有明确需要哪一种隔离，多 Agent 反而会变慢：

- 每个 worker 都要重新读上下文
- token 成本会乘上 worker 数量
- 任务切分和结果合成需要额外沟通
- 多个 worker 修改同一批文件会产生冲突

所以本篇的判断原则是：

```text
先判断要隔离什么，再选择多 Agent 形态。
```

---

## 二、Subagents：当前 session 内的隔离 worker

【官方事实】Subagents 是专门处理特定任务的 AI assistants。它们在自己的 context window 中工作，拥有自定义 system prompt、特定工具访问和独立 permissions。适合处理那些会让主会话塞满搜索结果、日志或文件内容的 side task。

换句话说，subagent 最核心的价值是：

```text
让脏活累活在另一个 context window 里完成，
主会话只接收结果摘要。
```

### 2.1 为什么 subagent 能省主 context

第 3 篇讲过，工具输出会进入 context。一次大范围代码搜索、全量测试日志、依赖树分析，很容易把主会话变得又大又杂。

Subagent 的机制是：

```text
主会话：提出任务
  -> subagent：开启自己的 context window
  -> subagent：搜索、读取、运行命令、整理结果
  -> 主会话：只收到摘要和必要结论
```

【官方事实】官方文档说明，每个 subagent 启动时有 fresh, isolated context window；它看不到你的完整 conversation history、已调用 skills 或已读取文件。Claude 会为它写一条 delegation message，总结任务。

【作者推导】这解释了为什么 subagent 适合“探索型、噪音大、最后只需要结论”的任务：

- 查一批错误日志，只要失败原因摘要
- 搜完整代码库，只要调用链结论
- 跑测试矩阵，只要失败清单
- 审查某模块，只要问题列表

如果任务需要主会话不断来回追问、共享大量中间上下文，subagent 反而会增加摩擦。

### 2.2 built-in subagents 与 custom subagents

【官方事实】Claude Code 内置了若干 subagents。下面只列出官方文档中职责较清晰、适合读者建立模型的几类；其他内置 helper agents 以当前 `/agents` 输出和官方文档为准，不在这里写成稳定承诺。

| subagent | 典型用途 | 工具倾向 |
| --- | --- | --- |
| Explore | 代码搜索、文件发现、代码库理解 | 只读工具，不能 Write / Edit |
| Plan | plan mode 中做代码库调研 | 只读工具 |
| general-purpose | 复杂、多步、可探索也可执行的任务 | 全工具 |

【官方事实】Explore 和 Plan 会跳过 CLAUDE.md 文件和父 session 的 git status，以保持研究更快、更低成本；其他 built-in 和 custom subagents 会加载这些内容。

【作者推导】这个细节很重要：不要以为所有 subagent 都完整继承主会话环境。Explore / Plan 更像轻量研究 worker，适合快速看代码，但如果某条项目规则必须影响它，例如“忽略 vendor 目录”，你需要在委托任务里明说。

Custom subagents 则适合把重复角色固化：

```markdown
---
name: code-reviewer
description: Reviews code for quality and security issues
tools: Read, Grep, Glob
model: sonnet
---

You are a code reviewer. Focus on correctness, security, test coverage,
and maintainability. Return actionable findings with file references.
```

【官方事实】Subagent 文件使用 YAML frontmatter 加 Markdown body。只有 `name` 和 `description` 必填；可配置 `tools`、`disallowedTools`、`model`、`permissionMode`、`mcpServers`、`hooks`、`skills`、`memory`、`background`、`isolation` 等字段。

### 2.3 subagent 的工具和权限边界

【官方事实】Subagents 默认继承主会话可用工具，包括 MCP tools。可以用 `tools` 做 allowlist，也可以用 `disallowedTools` 做 denylist。如果两者同时出现，会先应用 `disallowedTools`，再从剩余工具里解析 `tools`。

例如只读研究员：

```markdown
---
name: safe-researcher
description: Research agent with restricted capabilities
tools: Read, Grep, Glob, Bash
---
```

不允许写文件的 worker：

```markdown
---
name: no-writes
description: Inherits every tool except file writes
disallowedTools: Write, Edit
---
```

【作者推导】subagent 的工具限制是它比“直接让主会话做”更适合团队标准化的原因之一。你可以把某类角色限制成只读，把另一类角色允许改测试但不允许改生产代码，把数据库相关 MCP 只给专门的 validator。

但要注意：

- Subagents 不是越强越好，越强越要审权限。
- 背景 subagent 遇到需要权限提示的工具调用会自动拒绝。
- Subagents 不能再 spawn 其他 subagents，避免无限嵌套。

### 2.4 foreground 和 background 的差异

【官方事实】Subagents 可以前台或后台运行。前台 subagent 会阻塞主会话，权限提示会传给用户；后台 subagent 可以并发运行，但使用已经授予的权限，任何本来需要提示的工具调用会自动 deny。

【作者推导】这背后是一种交互权衡：

| 模式 | 优点 | 代价 |
| --- | --- | --- |
| foreground | 适合需要用户批准的高风险动作 | 主会话要等它结束 |
| background | 适合只读研究或长时间分析 | 不能弹出权限请求，容易因权限不足失败 |

因此，后台 subagent 适合：

- 只读调研
- 大范围搜索
- 日志整理
- 文档摘要

不适合：

- 需要频繁确认的写入
- 不确定是否要运行命令的任务
- 可能触发外部系统操作的任务

---

## 三、Agent Teams：多个独立 sessions 的协作系统

> 注意：Agent Teams 在官方文档中标注为 experimental，并且默认关闭。本文只描述截至 2026-05-19 的观察结果，不将其视为稳定行为保证。

以下内容主要用于理解 Agent Teams 的机制。生产或重要仓库中，建议先用单 session、subagents 或 worktrees 验证任务拆分方式，再考虑启用 Agent Teams。

【官方事实】Agent Teams 用来协调多个 Claude Code instances 一起工作。一个 session 作为 team lead，负责任务协调、分配和结果综合；teammates 是独立 Claude Code sessions，各自有自己的 context window，可以直接互相通信。Agent Teams 需要 Claude Code v2.1.32 或更高版本，并通过 `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1` 启用。

启用示意：

```json
{
  "env": {
    "CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS": "1"
  }
}
```

Agent Teams 的核心组件：

| 组件 | 作用 |
| --- | --- |
| Team lead | 创建团队、spawn teammates、协调任务、综合结果 |
| Teammates | 独立 Claude Code sessions，各自完成任务 |
| Task list | 共享任务列表，成员可 claim 和完成任务 |
| Mailbox | agents 之间传递消息 |

【官方事实】团队状态存储在本地，例如 team config 在 `~/.claude/teams/{team-name}/config.json`，task list 在 `~/.claude/tasks/{team-name}/`。这些运行时文件由 Claude Code 管理，不应手工编辑。

### 3.1 Agent Teams 和 subagents 的根本差异

两者都能并行，但架构不同。

| 维度 | Subagents | Agent Teams |
| --- | --- | --- |
| 所在范围 | 单个 session 内 | 多个独立 sessions |
| Context | 每个 subagent 自己的窗口，结果回主会话 | 每个 teammate 自己的窗口，完全独立 |
| 沟通方式 | 只向主 agent 返回结果 | teammates 可以直接互相消息 |
| 协调方式 | 主 agent 管理所有委托 | 共享任务列表 + lead 协调 + teammates 自协作 |
| 适合 | 只关心结果的 focused tasks | 需要讨论、挑战、协作的复杂任务 |
| token 成本 | 相对低，结果摘要回主会话 | 高，每个 teammate 都是独立 Claude instance |

【官方事实】官方文档明确说，subagents 向 main agent 回报；Agent Teams 中 teammates 通过共享任务列表和直接消息协调。Agent Teams 对顺序任务、同文件编辑、依赖很多的任务并不合适。

【作者推导】这就是 Agent Teams 的真正价值：不是“多几个 worker”，而是“让 worker 之间能互相校正”。

适合 Agent Teams 的任务：

- 设计一个新模块，让 UX、架构、测试、安全各看一面
- 根因不明的复杂 bug，让多个 teammates 验证不同假设
- PR 审查，让安全、性能、测试覆盖分别审
- 跨前端、后端、测试的功能设计，每人负责一层

不适合 Agent Teams 的任务：

- 改同一个函数
- 一步接一步的线性任务
- 只需要一次搜索和摘要
- 预算敏感的小任务
- 需要严格串行审批的变更

### 3.2 Agent Teams 的机制约束

【官方事实】Agent Teams 有已知限制：不支持带 in-process teammates 的完整 session resume；任务状态可能滞后；shutdown 可能较慢；同一 lead 一次只能管理一个 team；不支持 nested teams；lead 固定，不能把领导权转给 teammate。

【作者推导】这些限制决定了它更像“研究和协作工具”，还不是一个可以完全无人值守的项目经理。

使用时要给 lead 明确边界：

```text
Spawn 3 teammates:
- security-reviewer: 只看认证和权限
- performance-reviewer: 只看查询和缓存
- test-reviewer: 只看测试覆盖

先让他们各自调研，不要改文件。
等三个 teammate 都完成后，再由 lead 汇总共识和分歧。
```

好的 Agent Teams prompt 应该包含：

- 团队人数
- 每个 teammate 的角色
- 文件或模块边界
- 是否允许改文件
- 是否先 plan 再 implementation
- lead 什么时候汇总
- 质量门槛，例如测试、风险、证据

【官方事实】Agent Teams 还支持要求 teammates 先进入 plan mode，完成计划后由 lead 审批。lead 会自动做 approve / reject 判断，但你可以在 prompt 中给审批标准，例如“只有包含测试覆盖的计划才批准”。

【作者推导】这说明 Agent Teams 的安全和质量不只靠工具权限，还靠任务拆分设计。拆分越清楚，团队越像协作；拆分越模糊，团队越像多人同时踩一个文件。

---

## 四、Worktrees：文件系统隔离，不是协作系统

【官方事实】Git worktree 是一个独立工作目录，有自己的文件和分支，但共享同一个仓库历史和 remote。Claude Code 可以通过 `--worktree` / `-w` 创建独立 worktree 并在其中启动 session。

例如：

```bash
claude --worktree feature-auth
claude --worktree bugfix-123
```

默认会创建：

```text
.claude/worktrees/<name>/
```

并使用类似：

```text
worktree-<name>
```

的分支名。

【作者推导】Worktree 解决的是最朴素也最重要的问题：

```text
两个并行 session 不应该同时改同一个工作目录。
```

它不负责让 agents 协作，不负责分配任务，也不负责总结结果。它只确保文件改动隔离。

### 4.1 worktree 和 subagent / agent team 的关系

| 能力 | 解决什么 | 不解决什么 |
| --- | --- | --- |
| Subagent | context 隔离 | 文件改动冲突 |
| Agent Teams | 多 session 协作 | 默认不自动用 worktree 隔离每个 teammate |
| Worktree | 文件系统隔离 | 谁该做什么、怎么沟通 |

【官方事实】worktrees 文档说明，worktrees 隔离文件编辑，而 subagents 和 agent teams 协调工作本身。Subagents 可以通过 `isolation: worktree` 在自己的临时 worktree 中运行；并行 sessions 也可以各自用 worktree。

例如 custom subagent：

```markdown
---
name: refactor-worker
description: Refactors a scoped module in an isolated worktree
tools: Read, Grep, Glob, Edit, Bash
isolation: worktree
---

Only modify the module named in the task. Run focused tests before returning.
```

【作者推导】如果一个 subagent 只是做只读研究，不需要 worktree。如果它会改文件，尤其多个 subagents 可能并发改动，就应该考虑 worktree。

### 4.2 worktree 的配置和清理

【官方事实】Worktree 默认从 repository default branch / `origin/HEAD` 创建干净分支；如果没有 remote 或 fetch 失败，会回退到当前 local `HEAD`。如果希望从当前本地 `HEAD` 创建，可以设置：

```json
{
  "worktree": {
    "baseRef": "head"
  }
}
```

官方文档说明该设置只接受 `"fresh"` 或 `"head"`。

【官方事实】worktree 是 fresh checkout，所以主仓库里的 untracked files，例如 `.env`，不会自动存在。可以用 `.worktreeinclude` 复制 gitignored 文件，语法类似 `.gitignore`，只复制匹配且被 gitignore 的文件。

更合适的示意：

```text
.env.test
.env.local.example
config/test-secrets.json
```

不建议复制：

```text
.env.production
.env
prod.yaml
真实云账号凭据
真实数据库连接串
```

【作者推导】这里有一个安全取舍：worktree 里没有 `.env`，会导致测试缺配置；复制真实 `.env`，又可能把敏感文件带到更多目录。团队实践里最好准备 `.env.example`、`.env.test` 或测试专用配置。如果 `.env` 里是真实生产密钥，不要用 `.worktreeinclude` 扩散它，改用测试环境变量、mock 配置或本地开发专用凭据。

【官方事实】worktree cleanup 取决于是否有改动：没有未提交改动、未跟踪文件或新提交时，可自动清理；存在改动时会提示保留或删除；`--worktree` 配合 `-p` 的非交互式运行不会自动清理，需要用 `git worktree remove` 手动移除。

【作者推导】如果你让多个 Claude sessions 在 worktrees 里跑任务，最后一定要留一个复盘步骤：

```text
git worktree list
git status
git diff
git branch
```

否则很容易忘记某个 worktree 里还有未合并改动。

---

## 五、决策树：到底用哪一种

面对 subagents、Agent Teams 和 worktrees，最容易混淆的是“什么时候用哪一个”。可以先用一个最短决策树判断：

```text
任务是否需要并行？
  否 -> 单 session
  是 -> 是否需要文件系统隔离？
    是 -> git worktree + 多 session / worktree-isolated subagents
    否 -> 是否需要跨 session 长时间协作？
      是 -> Agent Teams（experimental）
      否 -> Subagents
```

再展开成工程判断：

| 场景 | 推荐 |
| --- | --- |
| 小改动、需要频繁来回 | 单 session |
| 大范围搜索、日志分析、只要摘要 | Subagent |
| 多个独立研究方向，需要最终合成 | 多个 subagents |
| 多个 session 由你手动协调，各自改不同任务 | Worktrees + 多 Claude sessions |
| subagent 会改文件，且可能和主会话冲突 | `isolation: worktree` |
| 多个 worker 需要互相讨论、挑战、共享任务状态 | Agent Teams |
| 全仓库机械迁移，多个隔离 worker 分片改动 | `/batch` 或 worktree-isolated subagents，后续单独研究 |

【作者推导】另一个更实际的问题是：谁来协调？

| 谁协调 | 选择 |
| --- | --- |
| 主会话 Claude 协调 | Subagents |
| 你自己协调多个窗口 | Worktrees + 多 sessions |
| Claude lead 协调一个团队 | Agent Teams |
| 需要每个 worker 打 PR | `/batch` 或 CI / GitHub Actions 路线 |

不要为了显得“多 Agent”而上 Agent Teams。只有当 teammates 之间的通信和 shared task list 真正带来价值时，它才值得额外 token 成本和协调复杂度。

---

## 六、组合使用时的新问题

前面已经分别解释了 subagents、Agent Teams 和 worktrees。真正容易出错的地方，通常发生在它们组合使用时。

### 6.1 subagent + worktree：两种隔离要叠加

【作者推导】Subagent 隔离的是“看见的信息”，worktree 隔离的是“写入的文件”。如果 subagent 只做调研，context 隔离已经够用；如果 subagent 会改文件，尤其多个 worker 并发改动，就要考虑 `isolation: worktree`。

更稳的组合是：

```text
只读调研 -> subagent
会改文件 -> subagent + isolation: worktree
多个会改文件的 worker -> 按目录/模块拆分 + worktree 隔离
```

### 6.2 Agent Teams + worktree：协作和文件隔离不是自动绑定

【作者推导】Agent Teams 解决 teammates 的沟通和任务协调，但不等于每个 teammate 天然拥有独立 worktree。只要多个 teammates 可能写文件，就仍然要明确文件边界、分支策略和 worktree 策略。

因此，Agent Teams prompt 里最好把这几句话写清楚：

```text
先只读调研，不要改文件。
如果需要改文件，先向 lead 报告文件范围。
不同 teammate 不要修改同一文件。
需要并行写入时，使用独立 worktree。
```

### 6.3 多 worker + permissions：成本和权限风险会乘法放大

【官方事实】官方文档多次提醒，多个 sessions、subagents 或 agent teams 会增加 token 使用。Agent Teams 的 token 成本随活跃 teammates 数量增长。

【作者推导】多 Agent 还会放大权限风险：

- 一个 worker 有权限运行命令，多个 worker 就可能并发运行命令
- 一个 worker 误改文件，多个 worker 就可能产生多个冲突
- 一个 worker 输出冗长结果，多个 worker 就可能把主会话摘要也变复杂

所以多 Agent 的安全策略要比单 session 更保守：

- 先只读研究，再允许写入
- 每个 worker 明确工具范围
- 写入型 worker 使用 worktree
- Agent Teams 明确角色、文件边界、退出条件
- 结果合成时要求证据，不只要结论

---

## 七、常见坑和修正

### 坑 1：把 subagent 当成万能加速器

错误认知：

```text
任务慢，就多开几个 subagents。
```

修正：

```text
只有当任务能独立拆分，且中间输出会污染主 context 时，subagent 才明显有价值。
```

### 坑 2：让多个 worker 改同一批文件

这会把并行变成冲突制造机。

修正：

- 按目录、模块、测试范围拆分
- 写入 worker 使用 worktree
- 同文件改动尽量单 session 串行完成

### 坑 3：忽略 Explore / Plan 的轻量特性

Explore / Plan 会跳过 CLAUDE.md 和 git status。如果某条规则必须影响它们，需要在委托 prompt 里明确写出来。

修正：

```text
请用 Explore 调研 src/auth，但忽略 vendor、dist 和 generated 目录。
```

### 坑 4：给 custom subagent 过宽工具权限

错误配置：

```markdown
tools: Read, Grep, Glob, Bash, Edit, Write
```

然后让它做只读审查。

修正：

```markdown
tools: Read, Grep, Glob
```

只读角色就保持只读。需要写文件时，再创建专门的写入角色。

### 坑 5：把 Agent Teams 用在顺序任务上

如果任务天然是：

```text
先改 A -> 再根据 A 改 B -> 最后测 C
```

Agent Teams 只会增加沟通成本。

修正：

- 顺序任务用单 session
- 大输出调研用 subagent
- 真正并行且需要交流时才用 Agent Teams

### 坑 6：worktree 里缺少本地配置

Worktree 是 fresh checkout，未跟踪的 `.env`、本地配置、虚拟环境不会自动出现。

修正：

- 用 `.env.example` 或测试配置
- 必要时用 `.worktreeinclude` 复制 gitignored 文件
- 不复制真实生产密钥
- 在每个 worktree 初始化依赖

### 坑 7：忘记清理 worktree 和 team

并行工作结束后，容易留下临时 worktree、branch、team runtime 状态。

修正：

```bash
git worktree list
git status
git branch
```

Agent Teams 结束时让 lead clean up；如果 tmux session 残留，按官方 troubleshooting 手动清理。

---

## 八、实践练习

【实践练习】下面的练习建议在测试仓库完成，不连接生产系统，不放真实密钥。每个练习至少保留一种可追溯产物：`/context` 输出、`/agents` 截图、git diff、`git worktree list` 输出、配置文件 diff、命令行输出或最小复现仓库。

### 练习 1：观察 subagent 的 context 隔离

**输入**：让 Claude Code 使用 subagent 对一个目录做大范围调研，例如“用 subagent 搜索 `src/` 下和认证相关的实现，返回入口文件、关键函数和风险点摘要”。调研前后分别运行 `/context`。  
**预期观察**：subagent 会在自己的 context 中读取和搜索大量内容，主会话主要接收摘要，而不是所有中间 Read / Grep 输出。  
**验证标准**：保存 `/context` 对比、会话截图或工具调用摘要。说明哪些中间内容没有直接进入主会话。

### 练习 2：创建一个只读 custom subagent

**输入**：在 `.claude/agents/` 下创建一个 `code-reviewer.md`，只允许 `Read, Grep, Glob`，然后要求它审查一个模块。  
**预期观察**：该 subagent 可以读取和搜索，但不能 Edit / Write。  
**验证标准**：保存 subagent 文件 diff、`/agents` 截图和会话输出。如果它尝试写文件，应记录权限失败或工具不可用表现。

### 练习 3：比较主会话和 subagent 的适用边界

**输入**：同一个问题分别让主会话直接调研、让 subagent 调研。例如“找出这个模块的错误处理路径”。  
**预期观察**：主会话方式更适合连续追问；subagent 方式更适合一次性搜索和摘要。  
**验证标准**：保存两次会话的工具调用摘要、`/context` 对比或截图，记录哪种方式更省主 context。

### 练习 4：用 worktree 跑两个隔离 session

**输入**：在测试仓库中分别运行：

```bash
claude --worktree feature-a
claude --worktree bugfix-b
```

让两个 session 修改不同文件。  
**预期观察**：两个 session 的改动位于不同 worktree，不会直接改主工作目录，也不会互相覆盖。  
**验证标准**：保存 `git worktree list`、两个 worktree 内的 `git status` 和 diff。结束后记录保留或清理策略。

### 练习 5：给 custom subagent 配置 `isolation: worktree`

**输入**：创建一个允许编辑测试文件的 subagent，并在 frontmatter 中加入 `isolation: worktree`。让它补一个测试。  
**预期观察**：subagent 在临时 worktree 中工作，改动和主会话工作目录隔离。若没有改动，worktree 可能自动清理；若有改动，应按提示保留或处理。  
**验证标准**：保存 subagent 文件、`git worktree list`、diff 或会话截图。说明 worktree 是否被保留。

### 练习 6：设计一个 Agent Teams 拆分方案

**输入**：不一定实际启用 Agent Teams，先写一份方案：选择一个复杂任务，设计 3 个 teammate 角色、各自负责的文件范围、是否允许写入、是否先 plan、lead 汇总标准。  
**预期观察**：方案能说明为什么需要 teammates 直接沟通，而不是普通 subagents。  
**验证标准**：保存设计文档，必须包含任务拆分、角色边界、共享任务列表预期、文件冲突规避、token 成本判断和 cleanup 策略。若实际启用 Agent Teams，保存启用配置和会话截图。

---

## 实践复盘建议

做完练习后，建议按三类整理：

- 与官方文档一致：subagent 是否有独立 context？Explore / Plan 是否只读？背景 subagent 是否自动 deny 需要提示的权限？worktree 是否隔离文件改动？Agent Teams 是否需要 experimental 环境变量？
- 与预期不同：是 subagent description 不够清晰、工具权限太宽或太窄、worktree 缺少 `.env`、两个 worker 改了同一文件、Agent Teams task status 滞后，还是 token 成本超出预期？
- 文档没有覆盖：保留 `/context`、`/agents`、`git worktree list`、配置 diff、会话截图或最小复现仓库，后续 Claude Code 版本升级后再复测。

这篇文章的核心结论可以压成一句话：

```text
Subagents 隔离 context，
Agent Teams 协调多个 sessions，
Worktrees 隔离文件改动；
多 Agent 的价值不在“开更多”，而在把上下文、协作和文件副作用分开管理。
```

---

## 参考资料

### 官方主参考

- [Run agents in parallel](https://code.claude.com/docs/en/agents)
- [Create custom subagents](https://code.claude.com/docs/en/sub-agents)
- [Agent Teams](https://code.claude.com/docs/en/agent-teams)
- [Worktrees](https://code.claude.com/docs/en/worktrees)
- [Configure permissions](https://code.claude.com/docs/en/permissions)

### GitHub / 社区补充

- [anthropics/claude-code](https://github.com/anthropics/claude-code)
- [subinium/awesome-claude-code](https://github.com/subinium/awesome-claude-code)

社区仓库适合观察别人如何组织 subagents、parallel workflows、worktree scripts 和团队实践，不用于替代官方文档对 Claude Code 行为的定义。凡是社区经验与官方文档冲突，以官方文档和本地实践观察为准。
