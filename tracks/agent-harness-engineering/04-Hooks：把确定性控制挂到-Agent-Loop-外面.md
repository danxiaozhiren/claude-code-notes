# Hooks：把确定性控制挂到 Agent Loop 外面

> 社区项目核查日期：2026-06-08
> 官方文档核查日期：2026-06-08
> 作者：wt
> 适合读者：个人开发者、团队技术负责人、想理解 Agent Harness 扩展点的工程师
> 本文定位：`learn-claude-code` 学习系列第 4 篇，承接第 3 篇的权限系统，解释为什么日志、权限、审计、后处理等确定性控制应该挂到 hook 点上，而不是继续塞进 agent loop。
> 实践建议：hook 实验建议放在临时目录；凡是会执行 shell、改文件、触发通知或调用外部服务的 hook，都要先用只读或 dry-run 版本验证。

---

## 先把边界说清楚

本文基于社区教学项目 `shareAI-lab/learn-claude-code`，不代表 Anthropic Claude Code 官方内部实现。

本文讨论的是 Claude Code-like Agent Harness 的通用 hooks 机制，不是在还原 Claude Code 源码。文中使用四类标签：

| 标签 | 含义 |
| --- | --- |
| 【官方事实】 | 来自 Claude Code 官方文档或官方公开材料 |
| 【社区教学实现】 | 来自 `shareAI-lab/learn-claude-code` 的 README 和 `code.py` |
| 【作者推导】 | 基于教学代码、官方边界和工程经验形成的解释 |
| 【动手观察】 | 读者可以本地复现的实验现象 |

特别说明：`learn-claude-code` 的部分 README 会讨论“真实 Claude Code”的实现细节。本文不把这些内容当成官方事实，只把它们当作社区作者的学习笔记和教学推导。涉及官方能力时，只以 Claude Code 官方文档为准。

前三篇已经建立了这条线：

```text
s01: agent loop 让模型能持续行动
s02: tool system 让模型从说变成做
s03: permission 在 handler 前约束行动边界
```

这一篇只问一个问题：

> 哪些确定性控制不该写进提示词，也不该写死在主循环里？

---

## 一、核心认知：Hook 是生命周期插口，不是又一个工具

工具是模型可以主动选择的动作。

Hook 不是。

Hook 是 harness 在固定生命周期点自动触发的控制逻辑。模型不需要“想起来调用 hook”，也不应该决定 hook 是否运行。

【官方事实】Claude Code 官方 Hooks reference 说明，hooks 是用户定义的 shell command、HTTP endpoint、LLM prompt 等处理器，会在 Claude Code 生命周期的特定事件上自动执行。官方文档把事件分成会话级、每轮对话级、工具调用级等多个节奏；其中 `PreToolUse` 在工具执行前触发，可以阻断工具调用；`PostToolUse` 在工具成功后触发。

【作者推导】这就是 hook 和 tool 的分界：

| 机制 | 谁触发 | 解决什么问题 |
| --- | --- | --- |
| Tool | 模型在某一轮选择 | 让 agent 做动作 |
| Permission | harness 在执行前判断 | 决定动作能不能发生 |
| Hook | harness 在生命周期点自动触发 | 挂载确定性控制、审计、后处理、通知 |
| Prompt | 模型读取后影响行为 | 告诉模型目标、约束和背景 |

如果一段逻辑必须“每次工具执行前都运行”，它就不应该靠 prompt 提醒模型，也不应该靠模型主动调用工具，而应该变成 `PreToolUse` hook。

---

## 二、为什么 s03 之后必须讲 hooks

s03 的权限系统已经把检查放在 handler 前：

```text
tool_use
  -> check_permission()
  -> handler
  -> tool_result
```

这很好，但很快会出现下一个问题。

如果你还想加：

```text
执行前写日志
执行前做权限检查
执行前通知审计系统
执行后检查输出大小
执行后自动格式化
执行后记录 metrics
停止前做总结
用户输入前做审计
```

最直接的写法是往 loop 里不断加代码：

```python
log_to_file(block)
check_permission(block)
notify_audit(block)
output = handler(**block.input)
check_large_output(output)
record_metrics(block, output)
```

【作者推导】这会把主循环从“agent 协议编排”变成“业务控制垃圾桶”。结果是：

| 问题 | 后果 |
| --- | --- |
| 每加一个横切能力都改 loop | loop 越来越难读，回归风险变高 |
| 权限、日志、审计混在一起 | 谁阻断、谁观察、谁后处理不清楚 |
| 每个工具复制一遍控制逻辑 | 新工具容易漏掉日志或权限 |
| 扩展顺序不明确 | 多个控制逻辑互相覆盖 |
| 很难按项目开关 | 不同仓库的规则都挤在同一个 loop |

s04 的价值不是“多了几个回调函数”，而是把横切控制从 loop 主干里移出去。

---

## 三、s04 的教学模型：注册表 + 触发器

【社区教学实现】`s04_hooks` 沿用 s03 的工具和权限逻辑，但把 `check_permission()` 从循环体内移到 `PreToolUse` hook 上。主循环不再直接调用权限函数，而是调用 `trigger_hooks("PreToolUse", block)`。

s04 的 hook 注册表很小：

```python
HOOKS = {
    "UserPromptSubmit": [],
    "PreToolUse": [],
    "PostToolUse": [],
    "Stop": [],
}
```

注册和触发也很小：

```python
def register_hook(event, callback):
    HOOKS[event].append(callback)

def trigger_hooks(event, *args):
    for callback in HOOKS[event]:
        result = callback(*args)
        if result is not None:
            return result
    return None
```

这两个函数背后的设计点是：

| 设计点 | 含义 |
| --- | --- |
| event name | 把生命周期点显式命名 |
| callback list | 一个事件可以挂多个控制逻辑 |
| ordered execution | 控制逻辑按注册顺序执行 |
| non-None result | 教学版用返回值表达阻断或强制继续 |

s04 只实现了 4 个事件：

| 事件 | 触发点 | 典型用途 |
| --- | --- | --- |
| `UserPromptSubmit` | 用户输入进入模型前 | 记录、注入上下文、审计 |
| `PreToolUse` | 工具 handler 执行前 | 权限、日志、阻断、请求确认 |
| `PostToolUse` | 工具 handler 成功后 | 输出检查、日志、后处理 |
| `Stop` | 模型不再请求工具、准备退出时 | 总结、清理、强制续跑 |

【作者推导】这 4 个点已经覆盖一个最小 agent cycle：

```text
用户输入
  -> UserPromptSubmit
  -> LLM
  -> tool_use?
     -> PreToolUse
     -> handler
     -> PostToolUse
     -> tool_result
  -> Stop
```

---

## 四、Hook 最关键的插入点：不改 loop 的控制面

s04 的 loop 变化很克制。

工具执行前：

```python
blocked = trigger_hooks("PreToolUse", block)
if blocked:
    results.append({
        "type": "tool_result",
        "tool_use_id": block.id,
        "content": str(blocked),
    })
    continue
```

工具执行后：

```python
output = handler(**block.input)
trigger_hooks("PostToolUse", block, output)
```

停止前：

```python
if response.stop_reason != "tool_use":
    force = trigger_hooks("Stop", messages)
    if force:
        messages.append({"role": "user", "content": force})
        continue
    return
```

【作者推导】hook 机制的工程价值在于把 loop 分成两层：

| 层次 | 职责 |
| --- | --- |
| 主循环 | 调 LLM、识别 tool_use、执行 handler、回填 tool_result、决定继续或停止 |
| hook 层 | 在固定生命周期点挂载权限、日志、审计、后处理、通知、策略 |

这能保护主循环的稳定性。后面新增“每次写文件后跑 formatter”“每次 bash 前记录审计日志”“输出过大时提醒模型”等能力，原则上不需要改 loop，只需要注册 hook。

---

## 五、官方事实：Claude Code 公开 hooks 模型给了哪些边界

【官方事实】Claude Code Hooks reference 把 hooks 定义为生命周期事件上的处理器，并公开了事件、matcher、handler 类型、JSON 输入输出、退出码、决策控制、异步 hook、安全建议等机制。

对本文最关键的是这几类：

| 官方公开机制 | 说明 | 对 harness 设计的启发 |
| --- | --- | --- |
| Hook lifecycle | 事件在会话、每轮、工具调用、压缩、worktree 等阶段触发 | hook 不只是工具前后，而是完整生命周期扩展点 |
| Matcher | `PreToolUse` / `PostToolUse` 等工具事件可以按工具名匹配 | hook 不必每次都运行，可以按工具过滤 |
| `if` condition | 可用权限规则语法进一步匹配工具参数 | 先粗匹配工具，再细匹配参数 |
| Command hook | shell 命令从 stdin 读取 JSON 输入 | hook 可以独立于 harness 主进程实现 |
| HTTP hook | 把事件 JSON POST 到 endpoint | 审计、通知、策略服务可以外置 |
| MCP tool hook | 可以调用已连接 MCP server 的工具 | hook 自身也能利用外部能力 |
| Prompt / agent hook | 可用模型或 agent 做单轮判断 | 某些复杂策略可以模型辅助，但要谨慎 |
| Decision control | 某些事件可以返回 deny、ask、allow、block 等决策 | hook 不只是观察，也能控制流程 |
| MCP matcher | MCP 工具按 `mcp__server__tool` 名称进入工具事件 | 外部工具也能被同一套 hook 匹配 |

这里有两个容易误解的点。

第一，hook 的 “allow” 不应被理解为可以绕过所有权限。官方 hooks 文档把 PreToolUse 与权限流放在同一条链路里；权限文档也说明 hook 不能绕过 deny/ask 规则。也就是说，hook 可以提供控制决策，但不应该成为绕过高优先级安全规则的后门。

第二，不是所有 hook 都应该阻断流程。有些 hook 只是观察型：

```text
记录日志
发通知
统计指标
检查输出大小
```

有些 hook 才是控制型：

```text
拒绝危险命令
要求用户确认
阻止继续下一轮
向模型注入纠错信息
```

把这两类混在一起，系统会很难调试。

---

## 六、深水区一：Hook 解决的是横切关注点

权限、日志、审计、metrics、通知、格式化都不是某一个工具独有的能力。

它们是横切关注点。

| 横切关注点 | 为什么不适合写进每个 handler |
| --- | --- |
| 权限 | 每个工具都复制一套规则，容易漏 |
| 日志 | 所有工具都要记录，和业务逻辑无关 |
| 审计 | 需要统一格式和统一落点 |
| 输出治理 | bash、read、MCP 都可能输出过大 |
| 通知 | 不应该让每个工具知道 Slack、Webhook、桌面通知 |
| 格式化 | 写文件后统一处理比每个写工具自己处理更稳 |

【作者推导】hook 的本质是控制面解耦：

```text
工具 handler: 完成动作
hook handler: 管理动作前后的确定性控制
```

这能让工具保持小而专注。比如 `write_file` 只负责写文件，至于写完后是否跑 formatter、是否记录审计、是否触发测试，不应该写死在 `write_file` 里。

---

## 七、深水区二：PreToolUse 和 Permission 的关系

第 3 篇已经讲了权限系统。第 4 篇讲 hooks，很容易出现一个问题：

> 权限到底是 permission 系统，还是 PreToolUse hook？

答案是：在一个教学 harness 里，权限可以作为 `PreToolUse` hook 实现；在更完整的系统里，权限与 hooks 通常是同一条执行前管线里的不同层。

s04 的做法是：

```text
PreToolUse:
  permission_hook
  log_hook
```

这让权限从主循环里移出来。

但需要保留一个安全原则：

```text
hook 可以参与权限决策
hook 不应该绕过更高优先级 deny/ask 规则
```

【作者推导】可以把执行前控制拆成三层：

| 层次 | 例子 | 谁更高优先级 |
| --- | --- | --- |
| 组织或项目策略 | 禁止 `git push`、禁止写 `.env` | 最高 |
| PreToolUse hook | 记录、阻断、询问、调用策略服务 | 中间 |
| handler guard | `safe_path()`、参数校验 | 最后一层 |

如果 hook 返回 “allow”，但组织策略明确 deny，这个动作仍然应该被拒绝。

---

## 八、深水区三：PostToolUse 不是“收尾”，而是结果治理

很多人把 `PostToolUse` 理解成 “after hook”，只用来打印一行日志。

这太浅。

`PostToolUse` 拿到的是：

```text
工具调用 + 工具输出
```

它可以做结果治理：

| 场景 | PostToolUse 可以做什么 |
| --- | --- |
| 输出过大 | 截断、提醒、写入附件、要求模型缩小范围 |
| 写文件成功 | 自动格式化、运行轻量检查、记录 changed file |
| 测试失败 | 抽取失败摘要，压缩后回填给模型 |
| MCP 返回敏感字段 | 脱敏或阻断继续传播 |
| bash 输出包含 secret | 警告、遮蔽、记录安全事件 |

s04 的 `large_output_hook` 只是最小例子：

```python
def large_output_hook(block, output):
    if len(str(output)) > 100000:
        print("large output")
```

【作者推导】真正重要的是这个 hook 拿到了 `block` 和 `output`。这意味着它可以同时知道：

```text
哪个工具产生了输出
输入参数是什么
输出有多大
是否需要后处理
```

结果治理不应该完全交给模型自由发挥。模型可以决定下一步，但 harness 需要先把结果变成可用、可控、可审计的 observation。

---

## 九、深水区四：Stop hook 需要防无限循环

s04 展示了一个简单 `Stop` hook：

```text
模型不再请求工具
  -> Stop hook 运行
  -> 如果返回内容，就把内容注入 messages
  -> loop 继续
```

这个机制很强，因为它可以让 harness 在模型准备停止时做最后检查。

例如：

```text
是否还有未完成 todo？
是否忘了总结？
是否有后台任务还没回收？
是否需要提示模型补充测试结果？
```

但它也很危险。

如果 Stop hook 每次都返回“继续修正”，agent loop 就可能永远停不下来。

【作者推导】生产级 Stop hook 至少要有：

| 保护 | 作用 |
| --- | --- |
| active flag | 防止 hook 触发后重入再次触发同一 hook |
| max retry | 最多强制续跑几次 |
| reason | 告诉模型为什么不能停 |
| idempotency | 重复运行不会产生额外副作用 |
| clear exit | 如果模型无法修正，能明确结束 |

这也是 hook 比普通函数调用更难的地方：hook 改变的是生命周期控制，不只是多执行一段代码。

---

## 十、深水区五：Matcher 决定 hook 成本和准确性

【官方事实】Claude Code Hooks reference 说明，工具相关事件可以用 matcher 按工具名匹配；matcher 可以是精确工具名、`|` 分隔的多个工具名，或者正则；还可以匹配 MCP 工具名，例如 `mcp__memory__.*`。

【作者推导】matcher 的意义不只是“少跑一点代码”。它决定了 hook 的成本和准确性。

| matcher | 适合场景 | 风险 |
| --- | --- | --- |
| `*` | 全量审计、全量日志 | 成本高，噪声大 |
| `Bash` | 只拦 shell | 文件工具绕过该 hook |
| `Edit|Write` | 写文件后格式化 | 不覆盖 bash 写文件 |
| `mcp__github__.*` | 记录 GitHub MCP 操作 | server 名变动会失效 |
| `mcp__.*__write.*` | 匹配外部写操作 | 命名约定不严时误判或漏判 |

一个好 hook 通常会分两段过滤：

```text
matcher: 粗过滤工具名
if condition / handler: 细看参数
```

例如：

```text
matcher = Bash
handler 检查 command 是否包含 git push
```

不要让每个 hook 都在所有工具调用上启动昂贵脚本。

---

## 十一、s20 的视角：机制很多，hook 点仍然很少

【社区教学实现】s20 把前面机制合并到一个完整 harness：dispatch、permission、hooks、todo、subagent、skills、context compact、memory、recovery、task、background、cron、team、worktree、MCP 等都回到一个 loop。

但 hook 点仍然很少：

```text
UserPromptSubmit
PreToolUse
PostToolUse
Stop
```

s20 的工具执行链路可以概括为：

```text
assemble_tool_pool()
  -> LLM
  -> for each tool_use
     -> trigger_hooks("PreToolUse", block)
     -> background? compact?
     -> call_tool_handler(...)
     -> trigger_hooks("PostToolUse", block, output)
     -> tool_result
  -> no tool_use
     -> trigger_hooks("Stop", messages)
```

【作者推导】这说明一个成熟 harness 不一定需要很多 hook 点。更重要的是 hook 点要稳定、语义清楚、覆盖关键生命周期。

| hook 点 | 为什么稳定 |
| --- | --- |
| 用户输入前 | 所有任务都从输入进入 |
| 工具执行前 | 所有真实行动都要经过 |
| 工具执行后 | 所有 observation 都从结果回来 |
| 停止前 | 每轮任务都要决定是否结束 |

后续 todo、compact、memory、background、MCP、team 都可以挂在这些稳定点附近，而不是不断重写 loop。

---

## 十二、机制解释：Hooks 为什么能让 harness 可扩展

把 hooks 放进 agent loop 后，整体结构变成：

```text
UserPromptSubmit
  -> LLM
  -> PreToolUse
  -> handler / MCP / background / compact
  -> PostToolUse
  -> tool_result
  -> Stop
```

Hooks 能让 harness 可扩展，是因为它给了工程师三个东西：

| 能力 | 解释 |
| --- | --- |
| 稳定生命周期点 | 不需要猜某段逻辑该插在哪里 |
| 统一事件输入 | hook 拿到工具名、输入、输出、会话等结构化上下文 |
| 可组合处理器 | 多个 hook 可以按顺序挂在同一个事件上 |

没有 hooks，新增一个横切能力通常意味着改主循环。

有 hooks，新增一个横切能力应该变成：

```text
选事件
写 handler
注册到 HOOKS
验证输入输出和阻断语义
```

【作者推导】hook 的边界是：它适合确定性控制，不适合把 agent 决策树搬到外部。

| 适合 hook | 不适合 hook |
| --- | --- |
| 拒绝危险命令 | 替模型规划任务 |
| 写审计日志 | 决定业务方案 |
| 写文件后格式化 | 根据复杂目标自由推理 |
| 输出过大时治理 | 代替工具系统做动作 |
| Stop 前做一致性检查 | 把 agent loop 变回 workflow engine |

如果 hook 开始承载大量业务决策，系统会回到“过程式工作流编排”的老路。

---

## 十三、最小代码理解：抓住 7 个代码点

读 s04 和 s20 时，先抓住这些点。

| 代码点 | 来自章节 | 看什么 | 为什么重要 |
| --- | --- | --- | --- |
| `HOOKS` | s04 / s20 | event name 到 callbacks 的映射 | hook 系统的注册表 |
| `register_hook()` | s04 / s20 | 如何注册扩展逻辑 | 扩展不改 loop |
| `trigger_hooks()` | s04 / s20 | 如何按事件触发回调 | hook 的统一入口 |
| `permission_hook()` | s04 / s20 | s03 权限如何变成 PreToolUse hook | 权限从硬编码变横切控制 |
| `log_hook()` | s04 / s20 | 每次工具调用前统一记录 | 观察型 hook 示例 |
| `large_output_hook()` | s04 / s20 | 工具输出后做大小检查 | 结果治理示例 |
| `Stop` hook | s04 / s20 | 没有 tool_use 时最后触发 | 退出控制和审计入口 |

最小结构图：

```text
HOOKS[event].append(callback)

agent_loop:
  trigger_hooks("UserPromptSubmit", query)
  response = LLM(...)
  if no tool_use:
      trigger_hooks("Stop", messages)
      return
  for block in tool_use:
      blocked = trigger_hooks("PreToolUse", block)
      if blocked: return tool_result(blocked)
      output = handler(...)
      trigger_hooks("PostToolUse", block, output)
      return tool_result(output)
```

理解了这张图，第 5 篇讲上下文工程时就容易理解：Todo、Skill、Compact、Memory 不是要把 loop 改成复杂流程，而是围绕这些稳定点管理 context。

---

## 十四、常见坑和修正

### 坑 1：把 hook 当成工具

如果让模型主动调用一个 `run_audit` 工具，就无法保证每次工具执行前都审计。

修正：必须自动发生的控制逻辑放进 hook，不放进 tool。

### 坑 2：每个 hook 都匹配所有事件

全量 matcher 会增加成本和噪声。

修正：用 matcher 先按工具或事件粗过滤，再在 handler 里看参数。

### 坑 3：让 hook 的 allow 绕过 deny

如果 hook 可以覆盖高优先级拒绝规则，就变成安全后门。

修正：deny / ask 规则优先级高于 hook allow；hook 的 allow 只能表示“该 hook 不反对”。

### 坑 4：PostToolUse 只打印日志

执行后阶段拿到了真实输出，只打印日志太浪费。

修正：把输出大小、敏感内容、格式化、轻量验证、结果摘要都作为候选后处理。

### 坑 5：Stop hook 没有重入保护

Stop hook 如果持续强制续跑，agent 可能永远不结束。

修正：加 active flag、最大续跑次数、明确 reason 和可退出路径。

### 坑 6：hook 里做太多业务决策

hook 如果开始替模型规划任务、选择方案，就会把 harness 变成硬编码 workflow。

修正：hook 只做确定性控制和边界治理，复杂推理仍交给模型。

### 坑 7：hook 失败没有策略

审计服务、HTTP endpoint、脚本都可能失败。

修正：区分 fail-open 和 fail-closed。安全相关 hook 失败应保守阻断；日志类 hook 失败可以记录后继续。

---

## 十五、动手实验

### 实验 1：确认权限从 loop 移到了 hook

**输入**：阅读 s03 和 s04 的 `agent_loop`，比较执行前检查的位置。

**预期观察**：s03 直接调用 `check_permission()`；s04 调用 `trigger_hooks("PreToolUse", block)`。

**验证标准**：说明为什么 s04 新增日志 hook 不需要改主循环。

### 实验 2：新增一个审计 hook

**输入**：在 s04 里新增 `audit_hook(block)`，把工具名和参数摘要写入临时日志文件。

**预期观察**：每次工具调用前都会记录日志。

**验证标准**：新增代码只包括 hook 函数和 `register_hook("PreToolUse", audit_hook)`。

### 实验 3：给写文件后加格式化提醒

**输入**：新增一个 `PostToolUse` hook，只匹配 `write_file` 或 `edit_file` 后输出提示。

**预期观察**：写文件工具完成后触发，读文件和 glob 不触发。

**验证标准**：解释为什么这比在 `write_file` handler 里硬编码更可扩展。

### 实验 4：测试大输出治理

**输入**：让工具返回超过阈值的大文本。

**预期观察**：`large_output_hook` 触发提醒。

**验证标准**：说明 PostToolUse 能拿到哪些信息，以及是否应该把完整大输出直接塞回 messages。

### 实验 5：做一个 Stop hook 保护

**输入**：写一个 Stop hook，当 session 中没有任何 tool_result 时提示模型补充说明，但最多触发一次。

**预期观察**：第一次停止前强制续跑，第二次不再重复。

**验证标准**：说明你用了什么状态避免无限循环。

### 实验 6：匹配 MCP 工具

**输入**：阅读官方 hooks 文档和 s19/s20 的 MCP 工具命名，写一个 matcher 设计：

```text
mcp__docs__.*
mcp__.*__write.*
```

**预期观察**：能解释这两类 matcher 分别会匹配什么。

**验证标准**：说明为什么 MCP 工具必须保留 server 命名空间。

---

## 自测问题

1. Hook 和 Tool 的根本区别是什么？
2. 为什么权限检查适合放在 `PreToolUse`，但不能让 hook allow 绕过 deny？
3. `PostToolUse` 为什么适合做输出治理？
4. 什么样的逻辑应该放进 hook，什么样的逻辑应该留给模型？
5. s04 的 `trigger_hooks()` 为什么返回非 `None` 就停止后续处理？这在教学版里简化了什么？
6. Stop hook 为什么需要重入保护？
7. matcher 粗过滤和 handler 细判断分别解决什么问题？
8. 如果你要“每次写文件后跑 formatter”，会选 Tool、Hook、Skill 还是 MCP？
9. 如果审计 hook 的 HTTP 服务挂了，应该 fail-open 还是 fail-closed？取决于什么？
10. s20 机制很多，但 hook 点很少，这说明了什么架构原则？

---

## 参考资料

### 官方主参考

- [Claude Code Hooks Reference](https://code.claude.com/docs/en/hooks)
- [Claude Code Permissions](https://code.claude.com/docs/en/permissions)
- [Claude Code Tools Reference](https://code.claude.com/docs/en/tools-reference)

### 社区教学项目

- [shareAI-lab/learn-claude-code](https://github.com/shareAI-lab/learn-claude-code)
- [s04_hooks](https://github.com/shareAI-lab/learn-claude-code/tree/main/s04_hooks)
- [s20_comprehensive](https://github.com/shareAI-lab/learn-claude-code/tree/main/s20_comprehensive)
