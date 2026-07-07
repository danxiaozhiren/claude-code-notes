# 多 Agent 协作：Subagent、Mailbox、Team Protocol、Worktree

> 社区项目核查日期：2026-06-09
> 官方文档核查日期：2026-07-07
> 作者：wt
> 适合读者：个人开发者、团队技术负责人、想理解 Agent Harness 多 Agent 协作机制的工程师
> 本文定位：`learn-claude-code` 学习系列第 7 篇，承接第 6 篇失败恢复与长任务，解释 Subagent、Task board、Mailbox、Team Protocol、Autonomous Agent、Worktree Isolation 如何一起支撑多 Agent 协作。
> 实践建议：本文涉及多线程队友、文件收件箱、任务自动认领和 git worktree。实验建议放在临时 clone 或一次性测试仓库中，不要在真实项目主工作区直接跑 worktree 删除、分支删除或写文件实验。
> 配套实验：[AHE-012、AHE-013、AHE-014](../../labs/agent-harness-engineering/00-Agent-Harness-实验手册.md)。

---

## 先把边界说清楚

本文基于社区教学项目 `shareAI-lab/learn-claude-code`，不代表 Anthropic Claude Code 官方内部实现。

本文讨论的是 Claude Code-like Agent Harness 的通用多 Agent 协作机制，不是在还原 Claude Code 源码。文中使用四类标签：

| 标签 | 含义 |
| --- | --- |
| 【官方事实】 | 来自 Claude Code 官方文档或官方公开材料 |
| 【社区教学实现】 | 来自 `shareAI-lab/learn-claude-code` 的 README 和 `code.py` |
| 【作者推导】 | 基于教学代码、官方边界和工程经验形成的解释 |
| 【动手观察】 | 读者可以本地复现的实验现象 |

特别说明：`learn-claude-code` 的部分 README 会讨论“真实 Claude Code”的实现细节。本文不把这些内容当成官方事实，只把它们当作社区作者的学习笔记和教学推导。涉及官方能力时，只以 Claude Code 官方文档为准。

第 6 篇讲的是长任务怎么不断掉：

```text
失败可恢复
任务可持久化
慢操作可后台化
未来事件可投递
```

这一篇进入更复杂的问题：

> 多个 agent 同时工作时，靠什么避免互相污染、重复劳动和覆盖文件？

---

## 一、核心认知：多 Agent 协作不是“多开几个模型”

很多人第一次看多 Agent，会把它理解成：

```text
开 3 个模型，并行干活。
```

这只说对了一半。

真正难的是并行之后的工程问题：

| 问题 | 如果不处理会怎样 |
| --- | --- |
| 上下文污染 | 子任务搜索出的日志、文件、失败路径塞满主会话 |
| 工作重复 | 两个 agent 同时做同一个任务 |
| 文件冲突 | 两个 agent 同时改同一批文件 |
| 状态不可见 | 队友做完了，但 lead 不知道 |
| 协议混乱 | 队友要审批、要停机、要交付，但消息只是自然语言 |
| 权限失控 | 子 agent 获得了不该有的工具或写权限 |

【作者推导】所以多 Agent Harness 的核心不是“并行”，而是三种隔离加一种协作：

| 机制 | 解决什么 |
| --- | --- |
| Context Isolation | 子任务不要污染主上下文 |
| Coordination State | 谁在做什么、谁被阻塞、谁完成了 |
| Filesystem Isolation | 并行编辑不要互相覆盖 |
| Communication Protocol | 状态、请求、审批、结果要可路由、可追踪 |

这四件事分别对应本文的几个关键词：

```text
Subagent: 上下文隔离
Task board: 协作状态
Mailbox: 异步通信
Team Protocol: 类型化请求/响应
Autonomous Agent: 自主认领任务
Worktree: 文件系统隔离
```

---

## 二、官方事实：Claude Code 公开文档里的多 Agent 边界

先看官方公开边界，再看社区教学实现。

【官方事实】Claude Code 的 Run agents in parallel 文档把并行工作分为 subagents、agent view、agent teams、dynamic workflows 等方式。文档说明，subagents 是单个 session 内的委托 worker，会在自己的上下文里完成子任务并返回 summary；agent view 标注为 research preview；agent teams 是多个协同 session，带共享任务列表和 agent 间消息，由 lead 管理，并标注为 experimental 且默认关闭；worktrees 给每个 session 单独 checkout，避免并行会话编辑同一批文件。

【官方事实】Subagents 文档说明，subagent 是处理特定任务类型的专门助手。每个 subagent 有独立 context window、自定义 system prompt、特定工具访问和独立权限；Claude 会根据 subagent 的 description 决定何时委托。文档还说明，subagent 文件用 YAML frontmatter 配置，常见字段包括 `name`、`description`、`tools`、`model`、`permissionMode`、`maxTurns`、`skills`、`mcpServers`、`background`、`isolation` 等。

【官方事实】同一 Subagents 文档说明，subagent 启动时不会继承完整 Claude Code system prompt；它接收自身 system prompt 和基本环境信息。subagent 默认从主会话当前工作目录开始；如果要给 subagent 仓库隔离副本，可以设置 `isolation: worktree`。文档还说明，subagents 不能再递归生成 subagents。

【官方事实】Agent Teams 文档说明，teammate 有自己的 context window；生成时会加载普通 session 会加载的项目上下文，例如 `CLAUDE.md`、MCP servers 和 skills，并接收 lead 的 spawn prompt，但 lead 的 conversation history 不会传过去。文档还说明，teammates 之间可以自动投递消息、空闲时通知 lead、共享 task list，并按名字互相发消息。

【官方事实】Agent Teams 文档还明确了几个风险：teammates 初始继承 lead 的权限设置；agent teams token 消耗显著高于单 session；任务太小会让协调成本超过收益；两个 teammates 编辑同一文件会导致覆盖风险，因此需要拆分文件所有权。

【官方事实】Worktrees 文档说明，git worktree 是一个独立工作目录，有自己的文件和分支，共享同一个仓库历史和远程；Claude Code 支持 `--worktree` / `-w` 创建隔离 worktree，也支持 subagent 使用 `isolation: worktree`。文档还说明，可用 `.worktreeinclude` 复制 gitignored 配置文件；无改动的 worktree 可自动清理，有改动时应保留或显式删除。

这些官方事实给本文一个边界：

| 主题 | 官方公开事实能确认什么 | 本文不会声称什么 |
| --- | --- | --- |
| Subagent | 独立上下文、专门 prompt、工具/权限限制、summary 回传 | 不声称 s06 的 `task` 工具就是官方内部实现 |
| Agent Teams | experimental、默认关闭、lead + teammates、共享任务和消息 | 不把社区代码的文件 mailbox 写成官方实现 |
| Worktree | 独立 checkout 避免文件冲突，subagent 可用 worktree 隔离 | 不声称 s18 的 `.worktrees/` 路径和分支命名等于官方行为 |
| Task list | 多 Agent 协作需要可见任务状态 | 不把 s12 的 JSON 格式写成官方任务格式 |
| Permission | subagent/teammate 权限要受控 | 不把教学版权限冒泡细节写成官方事实 |

---

## 三、s06：Subagent 先解决上下文污染

【社区教学实现】s06 新增一个 `task` 工具，用来启动 subagent。

最小结构是：

```python
def spawn_subagent(description):
    messages = [{"role": "user", "content": description}]
    for _ in range(30):
        response = call_model(system=SUB_SYSTEM, messages=messages, tools=SUB_TOOLS)
        ...
    return final_summary
```

这里有几个关键点。

第一，subagent 使用新的 `messages`。

```text
主 agent messages: 用户目标、历史、工具结果、当前计划
subagent messages: 只有这次 description 起步
```

这就是 context isolation。子任务可以读很多文件、跑很多搜索、收集很多证据，但最后只把 summary 回给主会话。

第二，subagent 有自己的 `SUB_SYSTEM` 和 `SUB_TOOLS`。

```text
主 agent: 有 task 工具，可以 spawn subagent
subagent: 没有 task 工具，不能递归 spawn
```

【作者推导】这不是小细节，而是多 Agent 安全边界。没有递归限制时，模型可能无限拆任务：

```text
主 agent -> subagent -> subagent -> subagent -> ...
```

第三，subagent 仍然经过 hooks 和权限检查。

s06 的 subagent 执行工具前也会走 `PreToolUse` hook。这表达一个重要原则：

```text
委托不等于绕过权限。
```

【作者推导】Subagent 的核心价值是“上下文减噪”，不是“多人协作”。它适合：

| 适合 subagent | 不适合只靠 subagent |
| --- | --- |
| 搜索代码库并总结 | 多个 worker 需要相互通信 |
| 分析日志并返回结论 | 多个 worker 要共同维护任务看板 |
| 生成独立调研结果 | 并行修改同一仓库多个文件 |
| 对某个模块做局部审查 | 需要长期 idle 和自主认领任务 |

所以 s06 只是多 Agent 的第一步：

```text
子任务隔离出去
结果摘要回来
主会话不被大量过程材料污染
```

---

## 四、s12：多 Agent 需要共享任务对象

第 6 篇已经讲过 s12 的 Task System。这里换一个角度看：它是多 Agent 协作的任务看板。

【社区教学实现】s12 的任务包含 `owner` 和 `blockedBy`：

```python
class Task:
    id: str
    subject: str
    status: str
    owner: str | None
    blockedBy: list[str]
```

`claim_task()` 的含义是：

```text
如果任务 pending
并且依赖完成
并且可以认领
就把 owner 设置为当前 agent
并进入 in_progress
```

【作者推导】多 Agent 必须有任务所有权。否则 lead 只能靠自然语言说：

```text
你做 A，你做 B，别重复。
```

这对模型太脆弱。更稳的是把任务状态写成共享对象：

| 字段 | 多 Agent 价值 |
| --- | --- |
| `status` | 当前任务是否还能被认领 |
| `owner` | 哪个 agent 正在负责 |
| `blockedBy` | 依赖没完成前不能开始 |
| `description` | 任务交付标准和背景 |

Task board 是 lead 和 teammate 的共同事实源：

```text
Lead: 创建任务、拆依赖、检查进度
Teammate: 查看任务、认领任务、完成任务
Harness: 根据状态阻止重复认领和过早启动
```

没有共享任务对象，多 Agent 很快会变成多个会话各说各话。

---

## 五、s15：Mailbox 让队友异步通信

s06 的 subagent 是一次委托，一次返回。

s15 开始进入 team：队友是后台线程，可以和 lead 异步通信。

【社区教学实现】s15 新增 `MessageBus`，每个 agent 有自己的 `.jsonl` inbox：

```python
BUS.send("lead", "reviewer", "please review auth")
BUS.read_inbox("lead")
```

教学版把消息写到：

```text
.mailboxes/<agent>.jsonl
```

读 inbox 时会读取并删除文件，表示消息已消费。

【作者推导】Mailbox 解决的不是“聊天”，而是异步交付。

| 场景 | 为什么需要 mailbox |
| --- | --- |
| teammate 做完了 | lead 不应靠轮询终端输出判断 |
| lead 改了任务要求 | teammate 下次工作前应能收到新指令 |
| teammate 请求审批 | lead 要能看到 request 并响应 |
| 多个 teammate 并行 | 每个 agent 需要自己的收件箱 |

s15 还新增 `spawn_teammate_thread()`：

```python
def spawn_teammate_thread(name, role, prompt):
    system = f"You are {name}, a {role}"
    threading.Thread(target=run).start()
```

teammate 有自己的：

```text
system prompt
messages
tools
inbox
thread lifecycle
```

它不是主会话里的一个函数调用结果，而是一个持续运行的 worker。

Lead 的主 loop 需要把 inbox 消息注入回 messages：

```text
MessageBus -> lead inbox -> messages.append(<inbox>...</inbox>) -> LLM
```

这和第 6 篇后台任务的 notification 是同一个设计原则：

> 异步结果必须回到 agent loop 的可见输入里。

如果 teammate 只是把结果写文件，不注入 lead 的上下文，lead 就无法基于它继续决策。

---

## 六、s16：Team Protocol 让消息从“聊天”变成“协议”

Mailbox 只解决投递问题，不解决语义问题。

例如 teammate 发送：

```text
I think we should change auth/session.py first.
```

这到底是：

```text
普通建议？
计划审批请求？
已完成结果？
阻塞报告？
停机响应？
```

如果只靠自然语言，lead 很难稳定处理。

【社区教学实现】s16 增加 `ProtocolState`：

```python
class ProtocolState:
    request_id: str
    type: str
    sender: str
    target: str
    status: str
    payload: str
```

并用 `request_id` 做请求响应匹配。

典型流程是：

```text
teammate submit_plan
  -> plan_approval_request(req_id)
  -> lead review_plan(req_id, approve=True/False)
  -> plan_approval_response(req_id)
  -> teammate 继续或调整
```

s16 还实现了 shutdown request：

```text
lead request_shutdown(teammate)
  -> shutdown_request(req_id)
  -> teammate shutdown_response(req_id)
  -> pending_requests[req_id] 更新状态
```

【作者推导】Team Protocol 的价值是把自然语言消息变成可路由事件。

| 没有协议 | 有协议 |
| --- | --- |
| 消息类型靠模型猜 | `type` 显式标记 |
| 请求和响应靠上下文记 | `request_id` 匹配 |
| 审批状态靠文字描述 | `pending/approved/rejected` |
| lead 不知道哪些请求未处理 | `pending_requests` 可查询 |
| 队友收到结果后不知道怎么解释 | response type 明确 |

这也是多 Agent 和普通聊天群的区别：

```text
多 Agent 不是让几个模型聊天，
而是让几个 worker 通过可追踪协议协作。
```

---

## 七、s17：Autonomous Agent 让队友空闲后还能继续找活

s15 的 teammate 更像“一次性工人”：spawn 后最多跑几轮，然后给 lead 发 summary。

s17 增加了自主队友：

```text
WORK -> IDLE -> WORK -> IDLE -> SHUTDOWN
```

【社区教学实现】s17 的 `idle_poll()` 做两件事：

```python
inbox = BUS.read_inbox(agent_name)
if inbox:
    return "work"

unclaimed = scan_unclaimed_tasks()
if unclaimed:
    claim_task(unclaimed[0]["id"], agent_name)
    return "work"
```

也就是说，teammate 空闲后不会立刻退出，而是：

| 检查顺序 | 目的 |
| --- | --- |
| 先看 inbox | lead 或其他 teammate 是否发来新消息 |
| 再扫任务看板 | 是否有 pending 且未被 owner 占用的任务 |
| 能认领就进入 WORK | 自动继续下一项 |
| 超时仍无事可做 | 退出并汇报 |

【作者推导】这一步让 team 从“lead 推任务”变成“队友拉任务”。

| Lead 推任务 | 队友拉任务 |
| --- | --- |
| lead 必须记得分配每一步 | 队友自己扫描可做任务 |
| lead 容易成为瓶颈 | task board 成为调度中心 |
| 队友做完就退出 | 队友空闲后继续寻找工作 |
| 适合小团队、短任务 | 适合长任务、多任务队列 |

但自主也带来风险。

如果任务粒度太粗，队友会长时间不汇报；如果任务粒度太细，协调成本会超过收益；如果 `claim_task` 没有 owner 检查，多个队友可能抢同一个任务。

所以 s17 的核心不是“agent 自己会干活”，而是：

```text
自主行动必须围绕共享任务看板和消息入口发生。
```

---

## 八、s18：Worktree 解决文件系统冲突，不解决协作本身

到 s17 为止，队友已经能：

```text
收消息
认领任务
提交计划
等待审批
继续找活
```

但还有一个硬问题：

```text
多个 agent 同时写同一个工作目录。
```

如果两个 teammate 都在主 checkout 里改文件，风险很直接：

```text
teammate A 改 src/auth.py
teammate B 也改 src/auth.py
谁覆盖谁？
怎么合并？
谁负责冲突？
```

【社区教学实现】s18 给 Task 增加 `worktree` 字段：

```python
class Task:
    ...
    worktree: str | None = None
```

然后用 `create_worktree(name, task_id)` 创建 git worktree，并把任务绑定到 worktree：

```text
git worktree add .worktrees/<name> -b wt/<name> HEAD
task.worktree = name
```

teammate claim 任务后，如果任务带 worktree，工具调用会在该 worktree 的 cwd 下执行：

```text
read_file / write_file / bash
  -> cwd = .worktrees/<task.worktree>
```

【作者推导】Worktree 的边界必须说清楚：

| Worktree 解决 | Worktree 不解决 |
| --- | --- |
| 文件编辑隔离 | 谁该做哪个任务 |
| 分支隔离 | 队友之间怎么通信 |
| 可保留结果给人 review | 计划审批和停机协议 |
| 并行修改不同区域 | 最终合并冲突 |

所以 worktree 不是 team protocol。它只是把文件系统冲突从“互相覆盖”降级为“后续合并”。

s18 的 `remove_worktree()` 也很重要：删除前会检查 uncommitted files 和 unpushed commits。

【作者推导】这体现一个工程原则：

```text
隔离区的清理必须 fail closed。
```

如果 worktree 有未提交文件或未推送提交，默认不应该静默删除。正确做法是：

```text
keep for review
merge later
explicit discard
```

---

## 九、把 s06、s12、s15-s18 串起来：三种隔离，一条回流

现在可以把第 7 篇串成一张机制图。

```text
                    ┌────────────────────┐
                    │       Lead         │
                    │ plan / assign /    │
                    │ synthesize         │
                    └─────────┬──────────┘
                              │
              spawn / message │ inbox / result
                              v
┌────────────────┐   ┌──────────────────┐   ┌────────────────┐
│ Task Board     │<->│ Teammate Agent   │<->│ Mailbox        │
│ status/owner   │   │ own context      │   │ typed messages │
│ blockedBy      │   │ own tools        │   │ request_id     │
└───────┬────────┘   └────────┬─────────┘   └────────────────┘
        │                     │
        │ bound worktree      │ cwd switch
        v                     v
┌────────────────────────────────────────┐
│ Worktree Isolation                     │
│ separate checkout / branch / review    │
└────────────────────────────────────────┘
```

【作者推导】多 Agent 协作至少要同时满足四个条件：

| 条件 | 对应机制 | 如果缺失 |
| --- | --- | --- |
| 子任务过程不污染主会话 | subagent fresh context | 主会话上下文爆炸 |
| 谁做什么可见 | task board + owner | 重复劳动或无人负责 |
| 队友结果可回流 | mailbox + inbox injection | lead 不知道发生了什么 |
| 并行编辑不覆盖 | worktree | 文件冲突直接发生在主工作区 |

最容易误解的是：这些机制不是层层替代，而是各管一段。

```text
Subagent 管上下文
Task 管工作所有权
Mailbox 管通信
Protocol 管消息语义
Autonomous loop 管持续找活
Worktree 管文件隔离
```

把它们混在一起，就会出问题。

例如只给每个 agent 一个 worktree，但没有 task board，它们仍然可能重复做同一个模块。只做 task board，但没有 worktree，它们仍然可能改同一份文件。只做 mailbox，但没有协议，lead 仍然要猜每条消息是什么意思。

---

## 十、几个容易踩的坑

### 1. 以为 subagent 就是 team

Subagent 是上下文隔离，不等于团队协作。

| subagent | team |
| --- | --- |
| 一次委托，返回 summary | 多个 teammate 持续运行 |
| 通常不需要 agent 间通信 | 需要 mailbox 和 protocol |
| 适合局部研究或审查 | 适合拆分、分配、跟踪和汇总 |

### 2. 让多个 agent 共享同一个工作目录

这是最危险的并行方式。即使每个 agent 都“很听话”，它们也可能根据过期上下文覆盖彼此修改。

并行写代码时至少要满足一个条件：

```text
要么严格按文件/模块分区，
要么每个 agent 使用独立 worktree。
```

### 3. 只有自然语言消息，没有协议字段

自然语言适合解释，协议字段适合路由。

一个 plan approval request 最少需要：

```text
type
request_id
sender
target
status
payload
```

否则 lead 很难判断哪些请求还 pending，队友也很难判断某条回复对应哪次请求。

### 4. 队友自主认领任务，但没有锁或 owner 检查

s17 的教学版已经检查 owner，但没有实现完整文件锁。真实工程里只靠“读一下再写”不够，可能有竞争条件。

【作者推导】如果要把这种机制用于生产级 harness，`claim_task()` 应该是原子操作：

```text
lock task file
reload task
verify pending + no owner + dependencies completed
write owner + status
unlock
```

### 5. Worktree 删除过于激进

worktree 是 agent 产出的隔离区。删除前必须确认：

```text
有没有 uncommitted changes？
有没有 unpushed commits？
有没有任务还引用它？
有没有人需要 review？
```

默认策略应该是保留可疑产物，而不是静默删除。

### 6. 忽略 token 和协调成本

多 Agent 会放大 token 消耗。官方 Agent Teams 文档也明确提醒，token 使用会随 teammate 数量增长。

【作者推导】多 Agent 适合这些场景：

| 适合 | 不适合 |
| --- | --- |
| 多假设排查 | 一个小 bug |
| 多模块并行审查 | 单文件小改动 |
| 明确可拆分的大迁移 | 需求还不清楚 |
| 互相验证的研究任务 | 只需要快速执行一个命令 |

---

## 十一、最小代码理解

下面只列最小代码点，目的是理解机制，不是复刻完整实现。

| 章节 | 最小代码点 | 机制含义 |
| --- | --- | --- |
| s06 | `spawn_subagent(description)` | 用 fresh messages 创建子上下文 |
| s06 | `SUB_TOOLS` 不含 `task` | 防止递归 spawn |
| s06 | `return result` | subagent 只把 summary 回给主会话 |
| s12 | `Task.owner` | 多 Agent 任务所有权 |
| s12 | `blockedBy + can_start()` | 依赖没完成前不能认领 |
| s15 | `MessageBus.send()` | agent 间异步投递 |
| s15 | `.mailboxes/<agent>.jsonl` | 每个 agent 独立 inbox |
| s15 | `spawn_teammate_thread()` | teammate 是持续运行 worker |
| s16 | `ProtocolState` | 请求、响应、审批有状态 |
| s16 | `request_id` | 响应能匹配回原请求 |
| s17 | `idle_poll()` | teammate 空闲后继续检查 inbox 和任务看板 |
| s17 | `scan_unclaimed_tasks()` | 从 push 模式变成 pull 模式 |
| s18 | `Task.worktree` | 任务和文件隔离区绑定 |
| s18 | `create_worktree()` | 为任务创建独立 checkout |
| s18 | `remove_worktree()` | 清理前检查未提交/未推送产物 |

可以把第 7 篇压缩成一段伪代码：

```python
lead.creates_tasks()
lead.spawns_teammates()

while teammate.alive:
    if inbox.has_message():
        handle_protocol_or_message()
    elif task_board.has_claimable_task():
        task = claim_task(owner=teammate)
        if task.worktree:
            cwd = worktree_path(task.worktree)
        work_on_task(cwd)
    else:
        idle_or_shutdown()

teammate.send_result_to_lead()
lead.synthesizes_results()
```

这段代码的重点是：

```text
不是每个 agent 都随便行动，
而是所有行动都围绕 task、mailbox、protocol、worktree 这些共享边界发生。
```

Lead 和 teammate 的最小时序可以写成：

| 顺序 | Lead | Teammate | 共享边界 |
| --- | --- | --- | --- |
| 1 | 拆任务、写 task board | 空闲等待 | `.tasks/` |
| 2 | spawn teammate | 启动独立上下文 | agent identity / tools |
| 3 | 发送请求或审批规则 | 读取 inbox | `.mailboxes/` |
| 4 | 等待 plan 或结果 | claim task / submit plan | `request_id` / `owner` |
| 5 | approve / reject / ask changes | 根据响应继续或停止 | Team Protocol |
| 6 | 汇总结果 | 在 worktree 中产出改动 | `Task.worktree` |

这张时序表说明：多 Agent 协作的核心不是“多开几个模型”，而是用任务、消息、协议和文件隔离把并行行动约束在可复盘的边界内。

---

## 十二、动手实验

以下实验建议在临时目录中完成。如果你本地没有 `learn-claude-code`，先 clone 到临时目录；不要把实验输出写进本文所在写作仓库。

本节实验对应实验手册编号：AHE-012、AHE-013、AHE-014。

### 实验 1：观察 s06 的 context isolation

输入：

```text
Ask a subagent to inspect this repository and summarize the main files.
Then answer only with the subagent's summary.
```

预期观察：

| 现象 | 说明 |
| --- | --- |
| 出现 `[Subagent spawned]` | 主 agent 通过 `task` 工具启动子任务 |
| subagent 有自己的工具调用过程 | 子任务在独立 messages 中执行 |
| 主会话只看到最终 summary | 大量搜索过程没有进入主上下文 |

验证标准：

```text
subagent 的中间 messages 不会全部回填给主 agent；
主 agent 拿到的是子任务结论，而不是完整执行历史。
```

### 实验 2：用 s15 跑两个队友并检查 lead inbox

输入：

```text
Spawn two teammates:
- reviewer: review README.md for clarity
- tester: inspect whether there are tests
Ask both to send results to lead. Then check inbox.
```

预期观察：

| 现象 | 说明 |
| --- | --- |
| `.mailboxes/` 出现 inbox 文件 | MessageBus 使用文件收件箱 |
| 两个 teammate 独立运行 | 每个有自己的 system/messages |
| `check_inbox` 能看到结果 | 异步结果回到 lead |

验证标准：

```text
队友不是直接把结果打印给用户；
结果必须通过 lead inbox 注入后才能进入 lead 决策。
```

### 实验 3：用 s16 做计划审批

输入：

```text
Spawn a teammate to propose a plan for refactoring a small module.
Ask it to submit the plan for review before editing.
Then approve or reject the plan.
```

预期观察：

| 现象 | 说明 |
| --- | --- |
| 出现 `plan_approval_request` | 消息有明确类型 |
| 出现 `req_xxxxxx` | 请求可追踪 |
| `review_plan` 后状态变化 | lead 可以审批或拒绝 |

验证标准：

```text
审批不是靠一句自然语言“我同意”；
而是通过 request_id 和 response type 匹配。
```

### 实验 4：观察 s17 的自动认领

输入：

```text
Create three pending tasks with clear descriptions.
Spawn one autonomous teammate and let it idle.
Do not manually assign tasks.
```

预期观察：

| 现象 | 说明 |
| --- | --- |
| teammate 空闲后扫描任务 | `idle_poll()` 运行 |
| pending task 被 claim | owner 被设置 |
| teammate 进入 work 阶段 | pull 模式生效 |

验证标准：

```text
任务不是 lead 手动逐个推给 teammate；
teammate 能从 task board 自己找可做任务。
```

### 实验 5：用 s18 绑定任务和 worktree

输入：

```text
Create a task named "edit isolated file".
Create a worktree for that task.
Spawn a teammate to claim and work on it.
Then inspect the main checkout and the worktree directory.
```

预期观察：

| 现象 | 说明 |
| --- | --- |
| `.worktrees/<name>` 出现 | 文件系统隔离区创建 |
| task JSON 有 `worktree` 字段 | 任务绑定隔离区 |
| teammate 的读写在 worktree cwd | 主 checkout 不被直接改动 |
| 删除 worktree 前有状态检查 | 避免丢未提交产物 |

验证标准：

```text
并行编辑应该发生在隔离 checkout；
主工作区不应被 teammate 直接污染。
```

---

## 十三、自测问题

1. Subagent 和 Agent Team 的边界是什么？
2. 为什么 subagent 返回 summary，而不是把完整 messages 回填给主会话？
3. Task board 中 `owner` 字段解决什么问题？
4. Mailbox 和普通终端日志有什么区别？
5. 为什么 plan approval 需要 `request_id`？
6. Autonomous teammate 为什么要先查 inbox，再扫 task board？
7. Worktree 解决了什么问题，又没有解决什么问题？
8. 如果两个 teammate 都需要改同一个文件，应该靠 worktree、任务拆分、还是 lead 协调？为什么？

---

## 参考资料

官方文档：

- [Claude Code Docs: Run agents in parallel](https://code.claude.com/docs/en/agents)
- [Claude Code Docs: Create custom subagents](https://code.claude.com/docs/en/sub-agents)
- [Claude Code Docs: Orchestrate teams of Claude Code sessions](https://code.claude.com/docs/en/agent-teams)
- [Claude Code Docs: Run parallel sessions with worktrees](https://code.claude.com/docs/en/worktrees)
- [Claude Code Docs: Manage sessions](https://code.claude.com/docs/en/sessions)

社区教学项目：

- [shareAI-lab/learn-claude-code: s06_subagent](https://github.com/shareAI-lab/learn-claude-code/tree/main/s06_subagent)
- [shareAI-lab/learn-claude-code: s12_task_system](https://github.com/shareAI-lab/learn-claude-code/tree/main/s12_task_system)
- [shareAI-lab/learn-claude-code: s15_agent_teams](https://github.com/shareAI-lab/learn-claude-code/tree/main/s15_agent_teams)
- [shareAI-lab/learn-claude-code: s16_team_protocols](https://github.com/shareAI-lab/learn-claude-code/tree/main/s16_team_protocols)
- [shareAI-lab/learn-claude-code: s17_autonomous_agents](https://github.com/shareAI-lab/learn-claude-code/tree/main/s17_autonomous_agents)
- [shareAI-lab/learn-claude-code: s18_worktree_isolation](https://github.com/shareAI-lab/learn-claude-code/tree/main/s18_worktree_isolation)
