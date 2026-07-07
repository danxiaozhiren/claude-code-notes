# 从最小 Agent Loop 开始：Claude Code-like Harness 的骨架

> 社区项目核查日期：2026-05-31  
> 官方文档核查日期：2026-05-31  
> 作者：wt  
> 适合读者：个人开发者、团队技术负责人、想理解 Agent Harness 的工程师  
> 本文定位：`learn-claude-code` 学习系列第 1 篇，从 `s01_agent_loop` 看 Claude Code-like harness 的最小骨架。  
> 实践建议：建议在临时目录运行教学代码，不要在重要项目目录里直接试验会执行 shell 命令的 demo。
> 配套实验：[AHE-001、AHE-002](../../labs/agent-harness-engineering/00-Agent-Harness-实验手册.md)。

---

## 先把边界说清楚

本文基于社区教学项目 `shareAI-lab/learn-claude-code`，不代表 Anthropic Claude Code 官方内部实现。

这句话要放在最前面，因为这组文章不是在“还原 Claude Code 源码”，而是在借一个社区教学项目理解 Agent Harness 的通用工程机制。这里的重点不是证明 Claude Code 内部就是这样写的，而是理解：

> 一个语言模型，怎样被 harness 包起来，变成能观察、行动、验证、继续行动的工程 agent？

本文会区分四类信息：

| 标签 | 含义 |
| --- | --- |
| 【官方事实】 | 来自 Claude Code 官方文档或官方公开材料 |
| 【社区教学实现】 | 来自 `shareAI-lab/learn-claude-code` 的 README 和 `code.py` |
| 【作者推导】 | 基于教学代码和工程经验形成的解释 |
| 【动手观察】 | 读者可以本地复现的实验现象 |

---

## 一、核心认知：最小 agent 不是一段 prompt，而是一个循环

很多人第一次理解 agent 时，会把注意力放在 prompt 上：

```text
只要 prompt 写得足够好，模型是不是就能自己完成任务？
```

这个理解只说中了一半。

Prompt 能影响模型的意图和风格，但 prompt 本身不能让模型看到文件系统，不能替模型执行命令，也不能把命令输出自动带回下一轮推理。没有 harness，模型最多只能“建议你运行某条命令”。有了 harness，模型才可以：

```text
提出下一步动作
  -> harness 执行动作
  -> harness 把真实结果写回 messages
  -> 模型基于新结果继续判断
```

【官方事实】Claude Code 官方文档把 Claude Code 描述为 agentic coding tool；官方“How Claude Code works”文档也把 Claude Code 放在 Claude 模型外侧，说明它提供工具、上下文管理和执行环境，把语言模型变成可行动的 coding agent。

【社区教学实现】`learn-claude-code` 的 s01 把这个结构压缩到一个最小例子：一个 `while True` 循环，一个 `bash` 工具，一个 `messages` 列表。模型如果请求工具，循环继续；模型不再请求工具，循环结束。

【作者推导】所以最小 agent harness 的骨架不是“更复杂的 prompt 模板”，而是一个能持续完成这三件事的运行时：

| 动作 | 谁负责 | 结果 |
| --- | --- | --- |
| 判断下一步要不要行动 | 模型 | 生成普通文本或工具调用 |
| 执行真实动作 | harness | 调用 bash、文件、API 等工具 |
| 把结果带回下一轮 | harness | 工具结果进入 `messages` |

这就是 Agent Loop 的起点。

---

## 二、s01 到底做了什么

`s01_agent_loop/code.py` 的教学目标很克制：先不做文件工具、权限系统、hooks、memory、subagent、MCP，只保留最小闭环。

它包含四个关键部件：

| 部件 | 在 s01 中的形态 | 作用 |
| --- | --- | --- |
| System prompt | `SYSTEM` | 告诉模型当前是 coding agent，工作目录在哪里 |
| Tool schema | `TOOLS` 里只有 `bash` | 告诉模型可以请求什么动作 |
| Tool handler | `run_bash(command)` | 真正执行 shell 命令并返回输出 |
| Agent loop | `agent_loop(messages)` | 调模型、执行工具、回填结果、继续循环 |

把细节去掉后，它的结构可以简化成这段伪代码：

```python
def agent_loop(messages):
    while True:
        response = llm(messages, tools)
        messages.append(assistant(response))

        if not response_requests_tool(response):
            return

        results = run_requested_tools(response)
        messages.append(user(results))
```

这段代码里最重要的不是 `while True` 本身，而是最后一行：

```python
messages.append(user(results))
```

工具结果不是打印给人看的临时日志，而是被重新放回对话历史，成为模型下一轮推理的输入。

【作者推导】这就是 agent 和普通“生成命令”的差别：普通模型把命令说出来就结束；harness 把命令跑完，把输出再喂回去，让模型继续做下一步判断。

---

## 三、`messages` 不是聊天记录，而是 agent 的工作现场

在普通聊天产品里，`messages` 容易被理解成“聊天历史”。但在 agent harness 里，它更像工作现场的连续观察记录。

一次最小任务可能变成这样：

```text
user: 请看一下当前目录有哪些 Python 文件
assistant: tool_use bash {"command": "find . -name '*.py'"}
user: tool_result "./s01_agent_loop/code.py\n./s02_tool_use/code.py"
assistant: 当前目录下有两个 Python 文件...
```

注意第三行。它虽然在 API 结构上经常作为 `user` 侧消息追加，但语义上并不是“用户又说了一句话”，而是 harness 把环境观察结果交还给模型。

【社区教学实现】s01 的 `agent_loop` 在执行 `run_bash()` 后，会构造 `tool_result`，并把它追加回 `messages`。这让模型下一轮能看到命令的真实输出。

【作者推导】从这个角度看，`messages` 至少混合了四类东西：

| 内容 | 例子 | 对 agent 的意义 |
| --- | --- | --- |
| 用户目标 | “修复这个测试” | 定义任务方向 |
| 模型决策 | “我要运行 pytest” | 记录行动选择 |
| 工具观察 | 测试输出、文件内容、命令错误 | 降低不确定性 |
| 模型总结 | “失败原因是...” | 形成阶段性解释 |

如果没有工具观察，模型只能猜。如果工具观察不进入下一轮，模型就无法基于现实反馈修正路线。

---

## 四、停止条件：模型不再请求工具时，harness 才结束

s01 的循环有一个关键判断：

```text
如果响应不是 tool_use，就结束。
```

这看起来简单，但它隐含了一个很重要的职责划分：

| 问题 | 决定者 |
| --- | --- |
| 下一步还要不要查文件、跑命令、继续验证？ | 模型 |
| 工具请求能不能被执行？ | harness |
| 工具执行结果如何进入下一轮？ | harness |
| 什么时候把最终答复交给用户？ | 模型不再请求工具时 |

【社区教学实现】s01 使用 `stop_reason` 判断是否继续工具轮。s20 的 README 则提醒，完整 harness 会更关注响应内容里是否实际出现 `tool_use` block，并在循环外加上 hooks、权限、压缩、恢复等机制。

【作者推导】这说明教学项目的主线不是“每章改一个更聪明的大脑”，而是不断把工程机制挂到同一个循环周围：

```text
s01: 模型请求工具 -> 执行 -> 回填
s02: 工具从 1 个变成多个
s03: 工具执行前加权限判断
s04: 工具前后加 hook
s08: LLM 前加上下文压缩
s11: 调用失败时加恢复策略
s19: 外部 MCP 工具接入同一个工具池
s20: 所有机制回到同一个 loop
```

换句话说，loop 是骨架，后面的机制是肌肉、感官、边界和恢复系统。

---

## 五、为什么“一个循环”能解释很多复杂能力

一个真实 coding agent 看起来很复杂：它能读项目、改文件、跑测试、查网页、调用外部工具、创建子任务、分派 subagent。

但这些能力可以先拆成两层：

```text
核心循环：模型提出行动，harness 执行行动，结果回填。
外围机制：工具、权限、hooks、memory、compact、subagent、MCP。
```

只要核心循环存在，外围机制就有明确的挂载位置。

| 机制 | 挂在哪里 | 解决什么问题 |
| --- | --- | --- |
| 工具系统 | LLM 可见的 `tools` 和 handler 分发 | 模型能做哪些动作 |
| 权限系统 | 工具执行前 | 哪些动作能直接做、要问、要拒绝 |
| Hooks | 用户输入、工具前后、停止前后 | 把确定性控制挂到 loop 外 |
| Context compact | LLM 调用前 | 防止历史和工具输出撑爆窗口 |
| Memory | system prompt 组装或工具读取 | 把长期有用信息带回来 |
| Subagent | 工具调用或任务分派处 | 用干净上下文处理旁路任务 |
| MCP | 工具池组装处 | 把外部系统接成可调用工具 |

【作者推导】这也是为什么学习 agent harness 时，先理解 s01 比先看完整系统更重要。完整系统的函数很多，但大多数复杂性都在回答同一个问题：

> 在模型下一次推理之前，harness 应该准备好什么？  
> 在模型请求动作之后，harness 应该允许什么、执行什么、记录什么、回填什么？

---

## 六、从 s01 到 s02：工具扩展不应该改主循环

s01 只有 `bash`。这足够证明 loop，但不适合真实开发。读文件要 `cat`，写文件要重定向，搜索文件要 `find`，编辑文件还要 shell 转义，既不稳定也不安全。

s02 的关键变化是增加四类工具：

```text
read_file
write_file
edit_file
glob
```

更重要的变化不是工具数量，而是工具分发方式。

【社区教学实现】s02 用 `TOOL_HANDLERS` 把工具名映射到 Python 函数。主循环不再硬编码 `run_bash()`，而是根据模型请求的 `block.name` 查 handler。

可以抽象成：

```python
handler = tool_handlers[tool_name]
output = handler(**tool_input)
```

【作者推导】这是 Agent Harness 的第一个工程分层：

| 不好的方向 | 更好的方向 |
| --- | --- |
| 每加一个工具就改 `agent_loop` | 每加一个工具只注册 schema 和 handler |
| loop 里堆 if/else | loop 只负责调度 |
| 工具逻辑和对话逻辑混在一起 | 工具实现独立演进 |

理解这个分层后，s19 的 MCP 就不难理解：外部工具也只是被规范化后接进同一个工具池。

---

## 七、最小代码理解：抓住 5 个代码点

读 s01 和 s02 时，不建议一开始陷进所有异常处理。先抓住这 5 个点。

| 代码点 | 看什么 | 为什么重要 |
| --- | --- | --- |
| `TOOLS` | 工具名、描述、输入 schema | 模型通过它知道可用动作 |
| `run_bash()` | handler 如何执行真实命令 | harness 是行动层，不是只生成文本 |
| `agent_loop()` | 调 LLM、检查工具、执行、回填 | 最小 agent loop |
| `tool_result` | 工具结果如何绑定 `tool_use_id` | 让模型知道哪个调用对应哪个结果 |
| `TOOL_HANDLERS` | 工具名到函数的映射 | 后续扩展工具的关键抽象 |

这 5 个点合起来，就是最小 harness 的结构图：

```text
Tool schema
  -> 模型选择工具
  -> Tool handler 执行
  -> tool_result 回填
  -> messages 更新
  -> 下一轮模型推理
```

【作者推导】如果你未来自己做一个 agent harness，第一版不必急着做 memory、workflow、agent team。先让这 5 个点跑通。只要结果能稳定回填，后面的机制才有基础。

---

## 八、不要把社区教学实现写成官方内部实现

这里需要再次强调边界。

【官方事实】Claude Code 是 Anthropic 的官方 agentic coding tool，公开文档描述了它能读代码库、编辑文件、运行命令，并通过工具、上下文管理和执行环境围绕 Claude 模型工作。

【社区教学实现】`learn-claude-code` 用 Python 教学代码重建了一个 Claude Code-like harness 的学习路径，从 s01 的最小 loop 逐步扩展到 s20 的完整组合。

【作者推导】这两者的关系应该这样写：

```text
不是：learn-claude-code 展示了 Claude Code 官方源码。
而是：learn-claude-code 用教学代码展示了 Claude Code-like harness 的通用机制。
```

这一点对技术写作很重要。社区项目的价值在于可读、可跑、可拆解；官方产品的价值在于真实能力和产品边界。两者可以互相校准概念，但不能互相替代来源。

---

## 九、常见坑和修正

### 坑 1：把 Agent Loop 理解成工作流编排

固定工作流通常是人写死步骤：

```text
先 A，再 B，再 C。
```

Agent Loop 更像：

```text
当前信息不够 -> 模型选择下一步 -> harness 执行 -> 根据结果再判断。
```

修正：不要在 loop 里塞满业务 if/else。业务能力应该变成工具、规则、hook 或 prompt context。

### 坑 2：以为工具越多越好

工具越多，模型选择空间越大，权限和安全成本也越高。s01 从一个 bash 开始，s02 才扩展到文件工具，是为了先看清楚“回填结果”这件事。

修正：新增工具前先问三个问题：

```text
这个工具是否比 bash 更稳定？
它的输入输出是否清晰？
它的副作用和权限边界是否可控？
```

### 坑 3：把 tool result 当日志

工具输出如果只是打印在终端，人看到了，模型没看到，agent 就无法继续基于结果推理。

修正：工具输出必须进入下一轮 `messages`，并和对应的工具请求建立关联。

### 坑 4：把社区教学代码当生产代码

s01 明确是教学 demo。它能执行 shell 命令，虽然有简单危险命令拦截，但不等于完整权限系统。

修正：在临时目录学习 s01；真正讨论边界时，进入 s03 权限系统和后续 hooks、worktree、MCP 权限章节。

---

## 十、动手实验

### 实验 1：观察最小 loop

**输入**：在临时目录运行 `python s01_agent_loop/code.py`，输入：

```text
List all Python files in this directory.
```

**预期观察**：模型应请求执行 shell 命令；命令结果返回后，模型再给出最终总结。

**验证标准**：能在终端输出里区分三件事：模型请求工具、harness 执行工具、工具结果影响最终回答。

### 实验 2：比较“只解释”和“需要行动”

**输入**：分别给 s01 输入两类任务：

```text
Explain what an agent loop is.
Create a hello.py file that prints hello.
```

**预期观察**：解释型任务可能直接返回文本；行动型任务更可能触发 bash。

**验证标准**：记录两次任务是否出现工具调用，并说明为什么一个任务不需要真实环境反馈，另一个任务需要。

### 实验 3：观察工具输出如何改变下一步

**输入**：在临时目录准备一个会失败的 Python 文件，再让 s01：

```text
Run this Python file and fix the problem if you can.
```

**预期观察**：模型会先运行命令，看到错误输出后再决定下一步。

**验证标准**：能指出“错误输出进入 messages 后，模型下一步判断发生了什么变化”。

### 实验 4：从 s01 切到 s02

**输入**：对 s01 和 s02 分别提出：

```text
Read code.py and summarize the agent_loop function.
```

**预期观察**：s01 只能通过 bash 间接读文件；s02 可以使用 `read_file` 工具。

**验证标准**：比较两者工具调用方式，说明专用工具为什么比让模型拼 shell 更清晰。

---

## 自测问题

1. 如果工具执行了，但结果没有追加回 `messages`，这个系统还算 agent loop 吗？
2. 为什么 s02 新增工具时要引入 `TOOL_HANDLERS`，而不是继续在 loop 里写 if/else？
3. `messages` 里哪些内容是用户目标，哪些内容是环境观察？
4. 为什么 s01 不适合在重要项目目录里直接运行？
5. 如果要给 s01 加权限系统，你会把检查放在 LLM 调用前、工具执行前，还是工具执行后？

---

## 参考资料

### 官方主参考

- [Claude Code overview](https://code.claude.com/docs/en/overview)
- [How Claude Code works](https://code.claude.com/docs/en/how-claude-code-works)

### 社区教学项目

- [shareAI-lab/learn-claude-code](https://github.com/shareAI-lab/learn-claude-code)
- [s01_agent_loop](https://github.com/shareAI-lab/learn-claude-code/tree/main/s01_agent_loop)
- [s02_tool_use](https://github.com/shareAI-lab/learn-claude-code/tree/main/s02_tool_use)
- [s20_comprehensive](https://github.com/shareAI-lab/learn-claude-code/tree/main/s20_comprehensive)
