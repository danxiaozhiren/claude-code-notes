# 失败恢复与长任务：Agent 如何不中途崩掉

> 社区项目核查日期：2026-06-09
> 官方文档核查日期：2026-07-07
> 作者：wt
> 适合读者：个人开发者、团队技术负责人、想理解 Agent Harness 稳定性机制的工程师
> 本文定位：`learn-claude-code` 学习系列第 6 篇，承接第 5 篇上下文工程，解释错误恢复、任务系统、后台任务和定时调度如何支撑长任务。
> 实践建议：本文涉及重试、后台命令、文件化任务和定时触发。实验建议放在临时目录，不要在真实项目根目录直接跑会写 `.tasks/`、`.memory/`、`.scheduled_tasks.json` 的教学代码。
> 配套实验：[AHE-006、AHE-008、AHE-009、AHE-010、AHE-011](../../labs/agent-harness-engineering/00-Agent-Harness-实验手册.md)。

---

## 先把边界说清楚

本文基于社区教学项目 `shareAI-lab/learn-claude-code`，不代表 Anthropic Claude Code 官方内部实现。

本文讨论的是 Claude Code-like Agent Harness 的通用稳定性机制，不是在还原 Claude Code 源码。文中使用四类标签：

| 标签 | 含义 |
| --- | --- |
| 【官方事实】 | 来自 Claude Code 官方文档或官方公开材料 |
| 【社区教学实现】 | 来自 `shareAI-lab/learn-claude-code` 的 README 和 `code.py` |
| 【作者推导】 | 基于教学代码、官方边界和工程经验形成的解释 |
| 【动手观察】 | 读者可以本地复现的实验现象 |

特别说明：`learn-claude-code` 的部分 README 会讨论“真实 Claude Code”的源码细节。本文不把这些内容当成官方事实，只把它们当作社区作者的学习笔记和教学推导。涉及官方能力时，只以 Claude Code 官方文档为准。

前五篇已经把 agent 的基本工作现场搭起来：

```text
Agent Loop
  -> Tool Use
  -> Permission
  -> Hooks
  -> Context Engineering
```

但这仍然不够。一个 agent 跑得越久，越容易遇到这些问题：

```text
模型输出被截断
上下文超限
API 临时失败
工具执行很慢
任务依赖没完成
用户希望明天或每小时再检查一次
```

这一篇只问一个问题：

> 长任务为什么不能只靠一次会话硬撑？

---

## 一、核心认知：长任务的关键不是“更聪明”，而是“状态不能只在当前轮里”

短任务可以依赖当前上下文：

```text
用户输入
  -> 模型思考
  -> 调工具
  -> 回答
```

长任务不行。长任务一定会遇到时间、错误和外部状态变化。

【作者推导】如果所有状态都只放在当前 messages 里，系统会有四类脆弱点：

| 脆弱点 | 表现 | 根因 |
| --- | --- | --- |
| 输出中断 | 模型刚写到一半就 `max_tokens` | 没有续写协议和恢复状态 |
| 上下文膨胀 | 历史、工具结果、文件内容塞满窗口 | 没有压缩和恢复入口 |
| 慢工具阻塞 | build/test/install 卡住整个 loop | 同步工具结果必须当场返回 |
| 未来任务丢失 | 用户希望定时检查，但会话结束就没了 | 没有会话外的任务和调度状态 |

所以 Agent Harness 需要把长任务拆成四层状态：

| 状态层 | 解决什么问题 | s11-s14 对应机制 |
| --- | --- | --- |
| Recovery State | 当前模型调用失败后怎么恢复 | s11 error recovery |
| Task State | 大目标拆成哪些可追踪任务 | s12 task system |
| Background State | 慢操作执行到哪里了 | s13 background tasks |
| Schedule State | 未来什么时候再把工作交给 agent | s14 cron scheduler |

这四层不是彼此替代，而是共同服务一个目标：

```text
当前轮中断了，工作还知道怎么继续；
当前会话变长了，关键状态仍能被找回；
当前工具没跑完，结果完成后还能回到 loop；
当前时间没到，未来触发时仍能生成一轮工作。
```

---

## 二、官方事实：Claude Code 公开文档里的稳定性边界

先看官方公开边界，再看社区教学实现。

【官方事实】Claude Code Sessions 文档说明，session 是绑定到项目目录的已保存对话；CLI 支持 `claude --continue`、`claude --resume` 和 `/resume` 等方式恢复会话。官方文档还说明，session 会持续保存到本地 transcript 文件中，可以在退出或 `/clear` 后回到之前的对话。

【官方事实】Agent SDK 的 sessions 文档进一步说明，session 持久化的是 conversation history，里面包含 prompt、工具调用、工具结果和模型响应；它不等于文件系统快照。需要回滚文件变化时，应使用 checkpointing。

【官方事实】Claude Code works 文档说明，每个对话是与当前目录绑定的 session；resume 会在同一 session ID 下继续追加消息，fork/branch 会复制历史到新的 session。该文档还说明，上下文窗口会包含对话历史、文件内容、命令输出、`CLAUDE.md`、auto memory、loaded skills 和系统指令；上下文满时会自动管理，必要时会清理旧工具输出或总结对话。

【官方事实】Interactive mode 文档说明，Claude Code 支持把 bash 命令放到后台运行：后台命令会异步执行，立即返回 background task ID；输出写入文件，Claude 可以之后读取；退出时后台任务会被清理，输出过大时也会被终止。

【官方事实】Hooks reference 和 hooks guide 公开了多个和恢复相关的生命周期点，例如 `PreCompact`、`PostCompact`、`SessionStart`、`Notification`、`StopFailure` 等。官方 guide 还给出一个模式：在 compact 后用 `SessionStart` hook 重新注入关键上下文。

【官方事实】Agent SDK Todo Lists 文档说明，Todo tracking 用于组织复杂工作和展示进度；截至 TypeScript Agent SDK 0.3.142 和 Claude Code v2.1.142，sessions 默认使用结构化 Task tools：`TaskCreate`、`TaskUpdate`、`TaskGet`、`TaskList`，而不是旧的 `TodoWrite`。

这些官方事实给第 6 篇一个清晰边界：

| 主题 | 官方公开事实能确认什么 | 本文不会声称什么 |
| --- | --- | --- |
| Session | 对话历史可恢复，和目录绑定 | 不声称社区教学代码的 session 存储就是官方实现 |
| Context | 上下文会自动管理和 compact | 不声称 s11 的 tail trimming 是官方 compact 算法 |
| Background bash | 长命令可以后台执行，有 ID 和输出检索 | 不声称 s13 的线程字典就是官方后台实现 |
| Hooks | compact、session、notification 等生命周期可挂控制逻辑 | 不把社区 README 的源码推导写成官方事实 |
| Task tools | 官方公开文档已说明 Task tools 和 TodoWrite 的迁移关系 | 不声称 s12 文件格式等于官方任务文件格式 |

---

## 三、s11：失败恢复不是“再试一次”，而是分类处理

【社区教学实现】s11 新增 `RecoveryState`，并把模型调用包在 `with_retry()` 和外层错误处理里。

最小结构可以理解成：

```python
state = RecoveryState()

try:
    response = with_retry(call_model, state)
except PromptTooLong:
    messages = reactive_compact(messages)
    continue

if response.stop_reason == "max_tokens":
    escalate_or_continue()
```

这里的关键不是“try/except”，而是把失败分成不同路径。

| 失败类型 | 教学版处理 | 为什么不能混在一起 |
| --- | --- | --- |
| 429 / 529 临时失败 | 指数退避，必要时 fallback model | 这类问题可能和请求内容无关，重试有意义 |
| prompt/context too long | reactive compact 后再试一次 | 原请求太大，盲目重试只会重复失败 |
| `max_tokens` 截断 | 先提高输出预算，再发 continuation prompt | 模型已经产出一部分，应该续写而不是重做 |
| 其他异常 | 写入错误消息并停止 | 不能把不可恢复错误伪装成成功 |

s11 的 `RecoveryState` 记录这些标志：

```python
class RecoveryState:
    has_escalated = False
    recovery_count = 0
    consecutive_529 = 0
    has_attempted_reactive_compact = False
    current_model = PRIMARY_MODEL
```

【作者推导】这个状态对象的意义是：恢复行为本身也需要状态。

没有恢复状态时，系统很容易出现三种坏结果：

| 坏结果 | 例子 |
| --- | --- |
| 无限恢复 | 每次超长都 compact，每次 compact 后又超长 |
| 重复做工 | 输出截断后从头再做，前面的工具结果和判断被浪费 |
| 错误吞掉 | API 失败后仍给用户一个像成功一样的回答 |

一个可维护的 agent runtime 至少要回答：

```text
这次失败是哪一类？
这类失败是否可恢复？
恢复最多尝试几次？
恢复后是否继续同一个任务？
恢复失败后怎么让用户看见真实状态？
```

这就是 s11 比“加 retry”更重要的地方。

---

## 四、s12：Task System 把大目标变成可恢复的工作图

第 5 篇讲过 Todo：它适合记录当前会话中的步骤。

s12 讲的是更硬一点的东西：任务图。

【社区教学实现】s12 用 `.tasks/` 下的 JSON 文件保存任务。每个任务至少包含：

```python
@dataclass
class Task:
    id: str
    subject: str
    description: str
    status: str
    owner: str | None
    blockedBy: list[str]
```

围绕这个数据结构，s12 提供了五个动作：

| 动作 | 作用 |
| --- | --- |
| `create_task` | 创建 pending 任务，可声明依赖 |
| `list_tasks` | 查看任务列表、状态、owner、依赖 |
| `get_task` | 查看某个任务完整 JSON |
| `claim_task` | 检查依赖，认领任务，进入 in_progress |
| `complete_task` | 完成任务，并报告被解锁的下游任务 |

核心状态机很小：

```text
pending
  -> claim_task
in_progress
  -> complete_task
completed
```

但依赖关系让它不再只是一个 todo list：

```text
任务 B blockedBy 任务 A
任务 A 没完成时，任务 B 不能 claim
任务 A complete 后，任务 B 被解锁
```

【作者推导】Task System 的价值不只是“记录计划”，而是把计划变成可恢复的工程对象。

| Todo 更像 | Task System 更像 |
| --- | --- |
| 当前工作清单 | 文件化任务数据库 |
| 模型提醒自己下一步 | harness 检查任务能不能开始 |
| 会话内状态 | 可跨轮次、跨进程、跨 agent 读取 |
| 简单顺序 | 可表达依赖和 owner |

这对长任务很关键。

如果用户说：

```text
把认证模块、数据库层、API 路由和测试一起重构。
```

模型在一轮上下文里“记住计划”是不够的。更稳的方式是：

```text
task_1: 梳理认证模块
task_2: 重构数据库访问层
task_3: 改 API 路由, blockedBy=[task_1, task_2]
task_4: 补集成测试, blockedBy=[task_3]
```

这样即使中途 compact、恢复、切换 agent，系统仍能问：

```text
哪些任务完成了？
哪些任务被阻塞？
当前 agent 正在负责哪个任务？
完成一个任务后解锁了哪些任务？
```

【作者推导】从 harness 设计看，Task System 是“长任务的骨架”。上下文可以忘掉细节，但任务文件不能随便丢。

---

## 五、s13：后台任务把“等待”从主循环里移出去

有些工具调用必须马上返回结果：

```text
read_file
list_tasks
get_task
```

有些工具调用可能很慢：

```text
npm install
pytest
docker build
make
deploy
```

如果所有工具都同步执行，agent loop 会被慢命令卡住：

```text
model -> bash("pytest")
  等 10 分钟
  才能回填 tool_result
  才能继续下一轮模型调用
```

【社区教学实现】s13 给 `bash` 增加 `run_in_background` 参数，并用启发式识别慢命令。慢操作被派发到后台线程，主 loop 立刻收到一个占位 tool_result：

```python
if should_run_background(block.name, block.input):
    bg_id = start_background_task(block)
    result = f"Background task {bg_id} started"
else:
    result = execute_tool(block)
```

后台任务有两张表：

```python
background_tasks[bg_id] = {"command": cmd, "status": "running"}
background_results[bg_id] = output
```

完成后，`collect_background_results()` 把结果变成通知，再注入下一轮 user message：

```text
<task_notification>
  <task_id>bg_0001</task_id>
  <status>completed</status>
  <command>pytest</command>
  <summary>...</summary>
</task_notification>
```

【作者推导】后台任务的本质是把同步协议改成异步协议：

| 同步工具 | 后台工具 |
| --- | --- |
| 调用后必须当场返回完整结果 | 先返回 background ID |
| 模型只能等待 | 模型可以继续做其他工作 |
| 结果就是 tool_result | 结果之后以 notification 回流 |
| 失败和超时容易卡死 loop | 可以加超时、取消、看门狗、输出上限 |

这里有一个很重要的设计点：

> 后台任务完成后，必须回到同一个 agent loop 的可见输入里。

如果后台线程只是把结果写到某个文件，但没有通知机制，模型不一定知道它完成了。反过来，如果通知没有 task ID，模型也无法把“哪个结果”关联到“哪个等待中的工作”。

s13 的 `<task_notification>` 是教学版简化，但它表达了一个通用 invariant：

```text
每一个异步动作，最终都要产生一个可追踪的 observation。
```

---

## 六、s14：Cron Scheduler 让未来时间也能生产工作

后台任务解决的是“现在已经开始但还没完成”的工作。

Cron Scheduler 解决的是另一类问题：

```text
现在还不能做，但未来某个时间要把工作重新交给 agent。
```

【社区教学实现】s14 增加 `CronJob`：

```python
@dataclass
class CronJob:
    id: str
    cron: str
    prompt: str
    recurring: bool
    durable: bool
```

它由四层组成：

| 层 | 职责 |
| --- | --- |
| CronJob 数据 | 记录什么时候触发、触发什么 prompt、是否持久化 |
| Scheduler thread | 每秒检查 cron 表达式是否匹配当前时间 |
| Cron queue | 调度线程只入队，不直接跑 agent |
| Agent loop injection | loop 消费队列，把 `[Scheduled] prompt` 注入 messages |

核心链路是：

```text
schedule_cron
  -> scheduled_jobs[job_id] = CronJob(...)
  -> cron_scheduler_loop 每秒检查
  -> cron_queue.append(job)
  -> agent_loop consume_cron_queue()
  -> messages.append("[Scheduled] ...")
```

【作者推导】这不是“给模型一个闹钟”这么简单。

Cron Scheduler 把时间变成了 agent 的输入源。它和用户输入、工具结果、后台通知本质上同级：

| 输入来源 | 进入 loop 的形式 |
| --- | --- |
| 用户主动输入 | user message |
| 工具执行结果 | tool_result |
| 后台任务完成 | task notification |
| 定时任务触发 | scheduled user message |

这对长任务很重要。很多工程任务不是连续执行，而是间歇执行：

```text
10 分钟后检查 CI
每天下午生成变更摘要
明天早上重新跑 flaky test
每小时检查一次部署状态，直到成功
```

没有调度系统时，agent 必须依赖用户回来提醒。加入调度系统后，harness 可以在未来把工作重新投递给 agent。

【作者推导】s14 最关键的设计不是 cron 语法，而是解耦：

```text
调度线程负责发现“该做了”
队列负责暂存“待交付”
agent loop 负责真正理解和执行
```

如果调度线程直接调用模型，会和当前会话、权限、上下文、后台任务抢状态。队列把这件事变成了可控交付。

---

## 七、把 s11-s14 串起来：长任务是一组状态机，不是一条长 prompt

现在可以把第 6 篇的四章合成一张图。

```text
                         ┌────────────────────┐
                         │  external events   │
                         │ user / cron / bg   │
                         └─────────┬──────────┘
                                   │
                                   v
┌──────────────┐    ┌────────────────────────┐    ┌──────────────┐
│ task files   │<-->|       agent loop       |<-->| recovery     │
│ .tasks/*.json│    │ messages + tools       │    │ state        │
└──────────────┘    └───────────┬────────────┘    └──────────────┘
                                │
                                v
                     ┌──────────────────────┐
                     │ tools                │
                     │ sync / background    │
                     └──────────┬───────────┘
                                │
                                v
                     ┌──────────────────────┐
                     │ observations return  │
                     │ tool_result / notif  │
                     └──────────────────────┘
```

【作者推导】稳定的 Agent Harness 不是靠“一次模型调用更长”，而是靠多个状态机相互配合：

| 状态机 | 输入 | 状态 | 输出 |
| --- | --- | --- | --- |
| Recovery | API 错误、stop_reason | retry count、compact flag、model | 继续、降级、停止 |
| Task | create/claim/complete | pending/in_progress/completed、依赖、owner | 可开始任务、被解锁任务 |
| Background | slow tool use | running/completed、bg_id、output | task notification |
| Cron | 时间匹配 | scheduled_jobs、last_fired、queue | scheduled message |

这解释了为什么“长任务不中途崩掉”不是单点能力。

它需要至少四个工程原则：

1. 失败可分类：不是所有错误都能重试，也不是所有错误都该停止。
2. 进度可持久化：大目标不能只藏在模型上下文里。
3. 等待可异步化：慢工具不能绑死主循环。
4. 未来可投递：到时间后必须能生成新的可处理输入。

---

## 八、几个容易踩的坑

### 1. 把 retry 当成错误恢复

retry 只能处理临时失败。

| 问题 | retry 是否有用 |
| --- | --- |
| 429 rate limit | 有用，但要退避 |
| 529 overload | 有用，但要限制次数和考虑 fallback |
| prompt too long | 盲目 retry 没用 |
| 权限拒绝 | retry 没用，要改变请求或问用户 |
| 工具参数错误 | retry 没用，要让模型自纠 |

错误恢复的第一步是分类，而不是循环。

### 2. 把 Task 当成 Todo

Todo 是“当前步骤可见化”。

Task System 是“工作对象持久化”。

如果只是提醒模型别忘了步骤，用 Todo 足够。如果任务需要依赖、owner、跨轮次恢复、跨 agent 协作，就需要 Task。

### 3. 后台任务只有执行，没有回流

最危险的后台实现是：

```text
启动一个线程
把 output 写到文件
不通知模型
```

这样模型看不到结果，用户也不知道任务是否完成。后台任务必须有 ID、状态、结果摘要和注入路径。

### 4. 定时器直接调用模型

定时器如果绕过主 loop 直接调用模型，会绕过权限、上下文、日志、hooks 和任务状态。更稳的方式是：

```text
timer -> queue -> normal agent loop
```

这样未来触发的工作和用户输入、工具结果走同一套 harness。

### 5. 恢复后不更新系统状态

s11 中每轮工具结果后都会 `update_context()` 并重新组装 system prompt。这个动作很重要。

恢复、后台结果、任务完成都会改变真实状态。模型下一轮看到的 system/context 如果没有更新，就会基于旧世界继续行动。

---

## 九、最小代码理解

下面只列最小代码点，目的是理解机制，不是复刻完整实现。

| 章节 | 最小代码点 | 机制含义 |
| --- | --- | --- |
| s11 | `RecoveryState` | 恢复行为本身要有状态，防止无限重试和重复升级 |
| s11 | `with_retry()` | 只对临时错误做退避重试 |
| s11 | `reactive_compact()` | prompt/context 过长时改变输入，而不是盲目重试 |
| s11 | `stop_reason == "max_tokens"` | 输出截断要升级预算或续写，不要把截断当完成 |
| s12 | `Task` dataclass | 把大目标变成文件化任务对象 |
| s12 | `blockedBy` + `can_start()` | 用依赖关系约束任务启动 |
| s12 | `claim_task()` | 显式 owner 和 in_progress 状态，避免无人负责 |
| s12 | `complete_task()` | 完成后报告解锁任务，让下游工作可见 |
| s13 | `run_in_background` | 让模型或 harness 标记慢操作异步执行 |
| s13 | `background_tasks` / `background_results` | 用注册表记录后台生命周期 |
| s13 | `collect_background_results()` | 把异步结果转成 notification 回到 loop |
| s14 | `CronJob` | 把未来触发条件和 prompt 变成持久对象 |
| s14 | `cron_queue` | 调度和 agent 执行解耦 |
| s14 | `consume_cron_queue()` | 未来事件最终仍以 message 进入 agent loop |

可以把这四章压缩成一个最小伪代码：

```python
while True:
    inject_scheduled_messages()
    inject_background_notifications()

    response = call_model_with_recovery(messages, recovery_state)

    if response.needs_tool:
        for tool_use in response.tool_uses:
            if should_background(tool_use):
                start_background(tool_use)
            else:
                run_tool(tool_use)

    update_task_files_if_needed()
    rebuild_context_from_real_state()
```

这段伪代码的重点不是函数名，而是顺序：

```text
先把外部事件回流
再让模型基于当前真实状态决策
工具执行后再更新状态
下一轮重新装配上下文
```

恢复状态机可以压缩成：

| 事件 | 首选处理 | 状态变化 | 退出条件 |
| --- | --- | --- | --- |
| 临时网络/API 错误 | retry / backoff | 增加 retry 次数 | 达到上限后返回失败 observation |
| `max_tokens` | 提高输出预算或 continuation | 记录 continuation 次数 | 模型补完或达到续写上限 |
| prompt too long | reactive compact | 生成压缩后 messages | 压缩后重试成功或仍失败 |
| 后台任务完成 | 注入 notification | task 从 running 到 done/failed | 下一轮模型看到结果 |
| cron 命中 | 入队 scheduled prompt | job fired / rescheduled | 事件回到正常 loop |

这张表的重点是：恢复不是“再试一次”，而是根据失败类型改变输入、预算、状态或调度入口。

---

## 十、动手实验

以下实验建议在临时目录中完成。如果你本地没有 `learn-claude-code`，先 clone 到临时目录；不要把实验输出写进本文所在写作仓库。

本节实验对应实验手册编号：AHE-006、AHE-008、AHE-009、AHE-010、AHE-011。

### 实验 1：观察 s11 的恢复路径

输入：

```sh
python s11_error_recovery/code.py
```

然后设计一个容易产生大量输出的请求，例如让 agent 生成一份很长的文件或摘要。

预期观察：

| 现象 | 说明 |
| --- | --- |
| 出现 `max_tokens` 相关日志 | 输出预算触顶 |
| 第一次触顶后升级 max_tokens | 教学版先尝试扩大输出预算 |
| 多次触顶后发送 continuation prompt | 系统让模型续写，而不是从头开始 |

验证标准：

```text
不是每个错误都直接退出；
也不是每个错误都无限 retry；
恢复路径有次数和类型限制。
```

### 实验 2：用 s12 建一个有依赖的任务图

输入：

```text
Create three tasks:
1. inspect auth module
2. refactor database layer
3. update API tests, blocked by the first two
Then list tasks and try to claim the third one.
```

预期观察：

| 现象 | 说明 |
| --- | --- |
| `.tasks/` 下出现 JSON 文件 | 任务状态被写到会话外 |
| 第三个任务无法直接 claim | `blockedBy` 生效 |
| 前两个任务完成后第三个任务可开始 | 依赖被解锁 |

验证标准：

```text
任务不是只存在于模型回复里；
依赖检查由工具函数执行，不靠模型自觉遵守。
```

### 实验 3：让 s13 后台跑一个慢命令

输入：

```text
Run "sleep 5 && echo done" in the background, then continue listing tasks.
```

预期观察：

| 现象 | 说明 |
| --- | --- |
| 立即返回 background task ID | 主 loop 没有被 `sleep` 阻塞 |
| 几秒后出现 task notification | 后台结果回到 agent 可见输入 |
| notification 包含 task id、状态、summary | 异步结果可追踪 |

验证标准：

```text
慢命令完成后，模型能看见它完成了；
不是只在终端后台静默执行。
```

### 实验 4：用 s14 创建一次性提醒

输入：

```text
Schedule a one-shot reminder in 1 minute to list all tasks.
Then do nothing and wait.
```

预期观察：

| 现象 | 说明 |
| --- | --- |
| `.scheduled_tasks.json` 写入 durable job | 定时任务会话外持久化 |
| 调度线程出现 fire 日志 | 时间匹配后入队 |
| agent 空闲时自动处理队列 | 未来事件被投递回 loop |

验证标准：

```text
定时触发不是模型“记住了”；
而是 scheduler 生产事件，queue processor 把事件交还给 agent loop。
```

---

## 十一、自测问题

1. 为什么 prompt too long 不能靠普通 retry 解决？
2. `max_tokens` 截断时，为什么第一次可以升级输出预算，而不是立刻让模型总结？
3. Todo 和 Task System 的边界是什么？
4. 后台任务为什么必须有 notification 回流？
5. Cron Scheduler 为什么不应该绕过 agent loop 直接调用模型？
6. 如果一个后台任务失败了，你会把失败结果写成 tool_result、notification，还是直接丢弃？为什么？
7. 长任务恢复后，为什么要重新从文件、任务表、memory 等真实状态装配上下文？

---

## 参考资料

官方文档：

- [Claude Code Docs: Manage sessions](https://code.claude.com/docs/en/sessions)
- [Claude Code Agent SDK: Work with sessions](https://code.claude.com/docs/en/agent-sdk/sessions)
- [Claude Code Docs: How Claude Code works](https://code.claude.com/docs/en/how-claude-code-works)
- [Claude Code Docs: Interactive mode](https://code.claude.com/docs/en/interactive-mode)
- [Claude Code Docs: Hooks reference](https://code.claude.com/docs/en/hooks)
- [Claude Code Docs: Automate actions with hooks](https://code.claude.com/docs/en/hooks-guide)
- [Claude Code Agent SDK: Todo Lists](https://code.claude.com/docs/en/agent-sdk/todo-tracking)

社区教学项目：

- [shareAI-lab/learn-claude-code: s11_error_recovery](https://github.com/shareAI-lab/learn-claude-code/tree/main/s11_error_recovery)
- [shareAI-lab/learn-claude-code: s12_task_system](https://github.com/shareAI-lab/learn-claude-code/tree/main/s12_task_system)
- [shareAI-lab/learn-claude-code: s13_background_tasks](https://github.com/shareAI-lab/learn-claude-code/tree/main/s13_background_tasks)
- [shareAI-lab/learn-claude-code: s14_cron_scheduler](https://github.com/shareAI-lab/learn-claude-code/tree/main/s14_cron_scheduler)
