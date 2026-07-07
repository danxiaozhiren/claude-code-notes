# 权限系统：Agent 自主行动前必须有边界

> 社区项目核查日期：2026-06-08
> 官方文档核查日期：2026-07-07
> 作者：wt
> 适合读者：个人开发者、团队技术负责人、想理解 Agent Harness 安全边界的工程师
> 本文定位：`learn-claude-code` 学习系列第 3 篇，承接第 2 篇的工具系统，解释工具真正执行前为什么必须经过 runtime 权限管线。
> 实践建议：权限实验必须放在临时目录；不要在重要仓库里测试删除、覆盖、权限变更、部署触发等动作。
> 配套实验：[AHE-004、AHE-016](../../labs/agent-harness-engineering/00-Agent-Harness-实验手册.md)。

---

## 先把边界说清楚

本文基于社区教学项目 `shareAI-lab/learn-claude-code`，不代表 Anthropic Claude Code 官方内部实现。

本文讨论的是 Claude Code-like Agent Harness 的通用权限机制，不是在还原 Claude Code 源码。文中使用四类标签：

| 标签 | 含义 |
| --- | --- |
| 【官方事实】 | 来自 Claude Code 官方文档或官方公开材料 |
| 【社区教学实现】 | 来自 `shareAI-lab/learn-claude-code` 的 README 和 `code.py` |
| 【作者推导】 | 基于教学代码、官方边界和工程经验形成的解释 |
| 【动手观察】 | 读者可以本地复现的实验现象 |

特别说明：`learn-claude-code` 的部分 README 会讨论“真实 Claude Code”的实现细节。本文不把这些内容当成官方事实，只把它们当作社区作者的学习笔记和教学推导。涉及官方能力时，只以 Claude Code 官方文档为准。

第 2 篇已经解释了工具系统：

```text
模型输出 tool_use
  -> harness 找到 handler
  -> handler 执行动作
  -> tool_result 回填 messages
```

这一篇只问一个问题：

> Agent 自主行动前，权限边界应该在哪里生效？

---

## 一、核心认知：权限不是提醒模型，而是约束 runtime

很多人第一次写 agent 权限时，会把边界写进 prompt：

```text
你不能删除文件。
你不能访问敏感目录。
你不能执行危险命令。
```

这有用，但不够。

Prompt 只能影响模型“想做什么”，不能保证 runtime “允许什么”。真正的权限系统必须在工具执行前拦住动作。

【官方事实】Claude Code 官方权限文档明确区分了提示词指导和权限执行：提示词或 `CLAUDE.md` 会影响 Claude 尝试做什么，但不会改变 Claude Code 允许什么；授权或撤销访问要通过 `/permissions`、权限规则、权限模式或 PreToolUse hook。

【作者推导】这句话对 harness 工程非常关键：

```text
prompt 是意图层约束
permission 是执行层约束
```

如果模型输出了：

```text
tool_use: bash
input: {"command": "rm -rf /tmp/demo"}
```

权限系统看的不应该是“模型刚才是不是说自己会小心”，而应该是：

```text
工具名是什么？
参数是什么？
命中哪些规则？
能不能直接执行？
要不要问用户？
必须拒绝吗？
```

这就是权限系统和普通安全提示的区别。

| 层次 | 处理对象 | 失败后果 |
| --- | --- | --- |
| Prompt 约束 | 模型的行为倾向 | 模型可能忽略、误解或在复杂任务中漂移 |
| Tool schema | 模型能提出哪些结构化动作 | 只能限制动作形态，不能决定是否批准 |
| Permission | 即将执行的具体工具调用 | 可以在 handler 前阻断真实动作 |
| OS sandbox | 进程实际能访问什么资源 | 防止 harness 自身规则漏判或子进程绕过 |

---

## 二、为什么 s02 之后必须立刻讲权限

s02 有了多个工具：

```text
bash
read_file
write_file
edit_file
glob
```

这些工具让模型从“说”变成“做”。但只要工具能做事，就会出现风险差异。

| 工具 | 典型动作 | 风险 |
| --- | --- | --- |
| `read_file` | 读取文件 | 可能读到 secrets、密钥、私人资料 |
| `write_file` | 覆盖或新建文件 | 可能破坏用户代码或配置 |
| `edit_file` | 局部替换 | 可能误改关键逻辑 |
| `glob` | 搜索路径 | 可能扩大上下文或泄露目录结构 |
| `bash` | 执行 shell | 风险最大，可能删除、联网、安装、提交、推送、启动进程 |

【社区教学实现】s02 已经有 `safe_path()`，能阻止部分路径逃逸。但 s02 的 `bash` 仍然缺少更细的执行前判断。s03 的问题定义很明确：如果用户说“清理一下项目”，没有权限管线的 agent 可能执行破坏性命令。

【作者推导】权限系统出现的位置不是偶然的。Agent Harness 的学习顺序应该是：

```text
先有工具
  -> 才有真实行动
  -> 才需要权限
```

没有工具时，模型最多说错话。有了工具后，模型可能做错事。权限系统就是从“回答安全”进入“行动安全”的分界线。

---

## 三、官方事实：Claude Code 公开权限模型给了哪些边界

先看官方公开边界，再看社区教学实现。

【官方事实】Claude Code 官方权限文档说明，它支持细粒度权限规则、权限模式和 managed policies，用来控制 Claude Code 能访问和执行什么。

官方文档中几个对本文最重要的点：

| 机制 | 官方公开说明 | 对 harness 设计的启发 |
| --- | --- | --- |
| 工具风险分层 | 只读工具通常无需批准；Bash 和文件修改通常需要批准 | 权限不应一刀切，读、写、执行要分级 |
| `/permissions` | 可以查看和管理工具权限规则及来源 | 权限需要可观察，而不是藏在代码里 |
| allow / ask / deny | allow 免手动确认，ask 每次确认，deny 阻止使用 | 权限结果至少要能表达放行、询问、拒绝 |
| 规则顺序 | deny -> ask -> allow，先匹配者生效 | 最保守规则应该优先生效 |
| 权限模式 | `default`、`acceptEdits`、`plan`、`auto`、`dontAsk`、`bypassPermissions` 等 | 同一套工具在不同信任模式下可以有不同审批策略 |
| Read/Edit 规则 | 支持项目相对、home 相对、绝对路径、gitignore 风格模式 | 路径权限必须有明确锚点 |
| PreToolUse hook | 工具调用前运行，可拒绝、要求提示或放行，但不能绕过 deny/ask 规则 | 权限系统要给确定性扩展留插口 |
| MCP 工具 | MCP 工具需要显式权限，命名形如 `mcp__server__tool` | 外部工具要按来源命名和授权 |

这里要注意一个容易误解的点。

【官方事实】Read/Edit 权限规则会尽力覆盖内置文件工具和 Claude Code 能识别的 Bash 文件读写命令，但不覆盖任意子进程间接读写文件。官方文档也提示，如果要做 OS 级强制执行，需要 sandbox。

【作者推导】这说明权限有层级：

```text
permission rules: harness 解释并拦截工具调用
handler guards: 工具内部做参数和路径校验
OS sandbox: 操作系统级兜底
```

不要把其中任何一层当成全部。

---

## 四、s03 的教学模型：三道闸门

【社区教学实现】`s03_permission` 没有重写 agent loop，也没有重写工具系统。它只在工具执行前插入一个 `check_permission()`。

s03 的教学管线可以概括为：

```text
tool_use
  -> Gate 1: hard deny list
  -> Gate 2: permission rules
  -> Gate 3: user approval
  -> handler
  -> tool_result
```

三道闸门分别解决不同问题：

| 闸门 | 判断方式 | 适合拦什么 |
| --- | --- | --- |
| hard deny list | 字符串或固定模式命中 | 永远不该执行的命令，例如系统级破坏动作 |
| permission rules | 按工具名和参数判断 | 写出工作区、危险 shell、敏感路径 |
| user approval | 需要人确认 | 有风险但不一定错误的动作 |

s03 的核心结构可以压缩成：

```python
DENY_LIST = ["rm -rf /", "sudo", "shutdown"]

PERMISSION_RULES = [
    {
        "tools": ["write_file", "edit_file"],
        "check": lambda args: path_escapes_workspace(args["path"]),
        "message": "Writing outside workspace",
    },
    {
        "tools": ["bash"],
        "check": lambda args: looks_destructive(args["command"]),
        "message": "Potentially destructive command",
    },
]
```

然后在 handler 前调用：

```python
if not check_permission(block):
    results.append({
        "type": "tool_result",
        "tool_use_id": block.id,
        "content": "Permission denied.",
    })
    continue

output = TOOL_HANDLERS[block.name](**block.input)
```

【作者推导】这段代码最值得看的不是 deny list 写了哪些词，而是插入点：

```text
模型已经选择了工具
handler 还没有执行
```

权限检查必须在这两个事件之间发生。

如果检查放得太早，系统只看到用户自然语言，无法判断具体行动。比如“清理项目”可能是 `rm -rf build/`，也可能只是 `find . -name "*.tmp"`。

如果检查放得太晚，handler 已经执行，权限系统只能记录事故。

---

## 五、权限拒绝也要进入 messages

s03 在拒绝后没有直接崩溃，也没有默默吞掉工具调用，而是把拒绝结果包装成 `tool_result`。

这很重要。

```text
assistant:
  tool_use id=tool_1 name=bash input={"command":"rm -rf /tmp/demo"}

harness:
  check_permission -> deny

user:
  tool_result tool_use_id=tool_1 content="Permission denied."

assistant:
  解释被拒绝，选择更安全动作，或询问用户
```

【作者推导】权限拒绝也是 observation。它告诉模型：

```text
这个动作没有发生。
原因是权限边界。
下一步不能假设它已经完成。
```

如果拒绝后不回填 messages，模型可能出现两种坏行为：

| 坏行为 | 原因 |
| --- | --- |
| 假装已完成 | 模型看不到拒绝结果，以为工具调用成功 |
| 重复撞墙 | 模型不知道为什么失败，下一轮继续发同样请求 |

所以权限系统不只是“拦截器”，也是上下文工程的一部分。它把 runtime 的边界反馈给模型，让模型在下一轮能调整策略。

---

## 六、深水区一：deny、ask、allow 不是三个按钮，而是三个信任等级

很多权限系统一开始只有一个布尔值：

```text
allowed = True / False
```

这很快不够用。

因为真实工程里有三类动作：

| 动作类型 | 示例 | 合适结果 |
| --- | --- | --- |
| 明确安全 | `read_file("README.md")` | allow |
| 明确危险 | `bash("rm -rf /")` | deny |
| 有风险但可能必要 | `edit_file("src/auth.py")`、`bash("npm install")` | ask |

【官方事实】Claude Code 的公开权限规则也使用 allow、ask、deny，并且规则顺序是 deny -> ask -> allow。

【作者推导】为什么 deny 要优先？

因为 allow 常常是宽规则，deny 常常是窄边界。

例如：

```json
{
  "allow": ["Bash(npm run *)"],
  "deny": ["Bash(npm run deploy *)"]
}
```

如果 allow 先匹配，部署命令可能被宽规则放过。deny 优先才能表达：

```text
通常允许 npm run
但 deploy 永远不允许自动执行
```

这就是权限系统里的“最小特权”原则：宽松规则必须被更严格的局部规则覆盖。

---

## 七、深水区二：路径边界和行为边界不是一回事

s02 已经有 `safe_path()`。那为什么 s03 还需要权限？

因为 `safe_path()` 只能回答一个问题：

```text
这个路径是否还在工作区里？
```

它不能回答：

```text
这个动作是否应该发生？
这个文件是否敏感？
这个修改是否破坏性？
这个 shell 命令会不会联网、删除、提权、发布？
```

举例：

| 动作 | 是否在工作区 | 是否应该自动允许 |
| --- | --- | --- |
| 读 `README.md` | 是 | 通常可以 |
| 覆盖 `src/main.py` | 是 | 需要看模式，可能 ask |
| 修改 `.env` | 是 | 应该 deny 或 ask |
| 执行 `git status` | 是 | 通常可以 |
| 执行 `git push origin main` | 是 | 高风险，至少 ask，团队环境常 deny |
| 删除 `node_modules/` | 是 | 可能合理，但仍应 ask |

【作者推导】路径边界解决的是“空间范围”，行为边界解决的是“动作语义”。

```text
safe_path: 你能去哪里
permission: 你能在那里做什么
sandbox: 就算规则漏了，进程实际能碰到什么
```

三者不能互相替代。

---

## 八、深水区三：权限检查应该看 tool input，而不是用户原话

用户说：

```text
帮我清理一下项目里的无用文件。
```

这句话本身不危险，也不安全。真正需要判断的是模型接下来提出的动作：

```text
bash("find . -name '*.tmp' -delete")
```

或者：

```text
bash("rm -rf .")
```

这也是为什么权限检查不能只做在用户输入阶段。

【作者推导】一个可靠的权限管线应该至少拿到：

| 字段 | 用途 |
| --- | --- |
| `tool_name` | 区分读、写、执行、外部系统调用 |
| `tool_input` | 判断路径、命令、URL、服务名、参数 |
| `cwd` / workspace | 判断相对路径落点 |
| permission mode | 决定默认是 ask、allow 还是 deny |
| rule source | 解释为什么允许或拒绝 |
| user/session context | 决定本次确认是否可复用 |

s03 只用了 `block.name` 和 `block.input`，这是教学简化，但方向是对的：看即将执行的结构化行动，而不是看自然语言愿望。

---

## 九、深水区四：权限模式是“信任环境”的表达

权限规则决定“什么动作命中什么策略”，权限模式决定“默认信任程度”。

【官方事实】Claude Code 官方文档公开列出了多种权限模式，例如：

| 模式 | 粗略含义 | 适合场景 |
| --- | --- | --- |
| `default` | 标准行为，工具首次使用通常提示 | 日常交互 |
| `acceptEdits` | 自动接受工作区内编辑和常见文件系统命令 | 明确让 agent 做代码修改 |
| `plan` | 允许读取和只读探索，不编辑源文件 | 先理解项目、先出方案 |
| `auto` | 自动批准并做后台安全检查，官方标注为研究预览 | 更高自动化但仍需评估 |
| `dontAsk` | 未预先允许的工具自动拒绝 | 严格受控环境 |
| `bypassPermissions` | 跳过大多数权限提示 | 只适合隔离容器或 VM 等环境 |

【作者推导】权限模式不是“开发者懒得点确认”的开关，而是环境信任模型。

同一个工具调用，在不同环境下应该有不同默认策略：

| 环境 | 读文件 | 改代码 | 执行测试 | 推送代码 |
| --- | --- | --- | --- | --- |
| 个人临时 demo | allow | ask | ask | ask/deny |
| 个人项目日常开发 | allow | ask 或 acceptEdits | allow/ask | ask |
| 团队主仓库 | allow | ask | allow/ask | deny 或强审批 |
| CI 容器 | allow | allow 工作目录 | allow | deny |
| 生产服务器 | 最小化 | deny | deny/ask | deny |

权限模式的设计目标是让同一套 harness 能在不同信任环境里运行，而不是每换一个环境就重写工具。

---

## 十、从单 Agent 到多 Agent：权限会“冒泡”

权限系统在单 agent 里已经复杂。到了多 agent，问题变成：

```text
子 agent 需要执行高风险动作时，谁来批准？
```

【社区教学实现】s15 的教学代码用 `MessageBus` 模拟队友之间的文件收件箱。README 说明教学版省略了完整权限冒泡，但把“队友请求审批 -> Lead 处理 -> 回复队友”的流程作为真实系统需要解决的问题。

可以把它抽象成：

```text
teammate 想执行 tool_use
  -> teammate 本地发现需要审批
  -> 发 permission_request 给 lead
  -> lead 展示给用户或策略层
  -> 发 permission_response 给 teammate
  -> teammate 继续或拒绝
```

【作者推导】权限冒泡的本质是：执行者和批准者不一定是同一个 agent。

在团队式 harness 里，权限不能简单绑定到“当前 loop”。你还需要记录：

| 字段 | 原因 |
| --- | --- |
| requester | 哪个 agent 请求动作 |
| owner / lead | 谁有权批准 |
| tool call | 请求的具体动作 |
| workspace / worktree | 动作发生在哪里 |
| request id | 批准结果要能回到原请求 |
| timeout / cancellation | 长时间无人批准时怎么办 |

这就是为什么多 agent 权限不能靠一行 `input("Allow?")` 解决。

---

## 十一、从同一目录到 worktree：隔离也是权限的一部分

权限不只存在于“是否允许工具调用”。工作目录本身也是边界。

【社区教学实现】s18 用 git worktree 做隔离：为任务创建独立目录和分支，把任务绑定到 worktree；队友认领任务后，它的 `bash`、`read_file`、`write_file` 在对应 worktree 下执行。

s18 的最小机制包括：

```text
validate_worktree_name(name)
create_worktree(name, task_id)
bind_task_to_worktree(task_id, name)
remove_worktree(name, discard_changes=False)
keep_worktree(name)
```

其中两个点和权限关系最密切：

| 机制 | 权限意义 |
| --- | --- |
| `validate_worktree_name` | 防止路径穿越和非法目录名 |
| `remove_worktree(..., discard_changes=False)` | 有未提交改动时默认拒绝删除 |

【作者推导】worktree 不是传统意义上的 allow/deny 规则，但它改变了 agent 的行动半径。

```text
没有 worktree: 多个 agent 共享同一个工作目录，互相覆盖风险高
有 worktree: 每个任务在独立目录和分支里行动，破坏半径变小
```

这属于权限系统的“空间隔离”维度。权限规则回答“能不能做”，worktree 回答“在哪里做、影响谁”。

---

## 十二、从内置工具到 MCP：外部能力必须带来源进入权限系统

MCP 让工具池从本地 handler 扩展到外部服务。权限边界也随之扩大。

【官方事实】Claude Agent SDK 的 MCP 文档说明，MCP 工具需要显式权限；MCP 工具名遵循 `mcp__<server-name>__<tool-name>`；可以用 `allowedTools` 精确批准某个 server 或某些工具。官方文档还提示，对 MCP 访问优先使用 `allowedTools`，不要用过宽的 `bypassPermissions` 代替。

【社区教学实现】s19 的 `assemble_tool_pool()` 把外部 MCP 工具规范化成：

```text
mcp__docs__search
mcp__deploy__trigger
```

其中 `docs.search` 是只读，`deploy.trigger` 是破坏性动作。教学版只在 description 里标注 `(readOnly)` 或 `(destructive)`，没有做完整权限拦截。

【作者推导】MCP 权限至少要解决三个问题：

| 问题 | 为什么重要 |
| --- | --- |
| 工具来自哪个 server | 同名 `search` 可能来自 docs，也可能来自工单系统 |
| 工具动作是否只读 | 查询文档和触发部署不是同一类风险 |
| server 本身是否可信 | 外部服务有认证、网络、数据访问和副作用 |

所以 MCP 工具不能只作为“更多工具”进入 pool，还必须作为“带来源和风险标签的能力”进入权限系统。

---

## 十三、机制解释：权限系统为什么能约束自主行动

把前面的内容合起来，一条完整的权限链路是：

```text
用户目标
  -> LLM 选择工具
  -> harness 拿到 tool_name + tool_input
  -> 参数验证
  -> hard deny / ask / allow 规则
  -> hook / policy / user approval
  -> handler 或 MCP handler
  -> tool_result
  -> 下一轮模型
```

权限系统能生效，是因为它站在模型和真实行动之间。

| 如果缺少 | 会发生什么 |
| --- | --- |
| 缺少结构化工具调用 | 只能分析自然语言，无法判断真实动作 |
| 缺少执行前检查 | handler 已执行，权限只能事后记录 |
| 缺少 deny 优先 | 宽 allow 规则可能覆盖危险例外 |
| 缺少 ask | 风险动作只能全放或全禁，无法让用户授权 |
| 缺少结果回填 | 模型不知道动作被拒绝，可能假装完成 |
| 缺少来源命名 | MCP、subagent、worktree 场景里无法追责 |

【作者推导】权限系统的目标不是让 agent 变胆小，而是让 agent 能在明确边界里更大胆。

没有边界时，团队只能不信任 agent。有边界时，团队可以把更多动作交给 agent，因为高风险动作会被拦截、询问或隔离。

最小决策表可以这样记：

| 决策 | 触发条件 | handler 是否执行 | 回填给模型的结果 | 设计意图 |
| --- | --- | --- | --- | --- |
| `deny` | 明确危险、越界、违反团队规则 | 否 | 拒绝原因和替代建议 | 保护不可逆边界 |
| `ask` | 风险可接受但需要人确认 | 等待批准后再决定 | 批准或拒绝结果 | 把信任升级交给用户 |
| `allow` | 已知安全、低风险、在授权范围内 | 是 | 正常 `tool_result` | 让低风险动作自动化 |

这张表的关键不是三种词，而是默认方向：不确定时不要把动作送进 handler。

---

## 十四、最小代码理解：抓住 8 个代码点

读 s03、s15、s18、s19 时，先抓住这些点。

| 代码点 | 来自章节 | 看什么 | 为什么重要 |
| --- | --- | --- | --- |
| `DENY_LIST` | s03 | 永远拒绝的命令片段 | 展示 hard deny 的最低限度 |
| `PERMISSION_RULES` | s03 | 按工具名和参数匹配规则 | 权限判断要看结构化 input |
| `ask_user()` | s03 | 风险动作转为人工审批 | ask 不是失败，是信任升级 |
| `check_permission()` | s03 | deny -> rule -> ask -> allow | 权限管线的最小形态 |
| `agent_loop` 插入点 | s03 | handler 前调用权限检查 | 执行前阻断真实动作 |
| `MessageBus` | s15 | 队友之间传递结构化消息 | 多 agent 权限需要请求和响应通道 |
| `validate_worktree_name()` / `remove_worktree()` | s18 | 路径名校验和删除前检查 | worktree 是空间隔离和破坏半径控制 |
| `mcp__server__tool` | s19 | 外部工具命名空间 | MCP 权限必须知道工具来源 |

最小结构图可以写成：

```text
tool_use
  -> validate input
  -> permission decision
     -> deny: tool_result("Permission denied")
     -> ask: wait for approval
     -> allow: execute handler
  -> tool_result
  -> messages
```

理解了这张图，下一篇讲 hooks 时就很自然：hooks 不是替代权限，而是把确定性控制挂到权限和工具执行链路上。

---

## 十五、常见坑和修正

### 坑 1：把“别做危险操作”写进 prompt 就算权限

Prompt 是行为引导，不是执行边界。

修正：在 handler 前检查 `tool_name` 和 `tool_input`。高风险动作必须由 runtime 决策。

### 坑 2：只用 deny list

固定 deny list 能挡住少数明显危险命令，但挡不住变体、脚本、间接执行和业务风险。

修正：deny list 只作为最后的硬边界之一；还要有按工具、路径、命令、服务、模式的规则。

### 坑 3：只用 allow list

宽 allow 很容易误放危险子命令。例如允许 `Bash(npm run *)`，但 `npm run deploy` 可能触发生产发布。

修正：deny 优先，ask 其次，allow 最后。对宽规则保留更窄的拒绝例外。

### 坑 4：拒绝后不给模型反馈

如果拒绝没有进入 `messages`，模型下一轮不知道动作没发生。

修正：把拒绝包装成 `tool_result`，并保留 `tool_use_id`。

### 坑 5：把路径校验当成完整权限

`safe_path()` 只能限制路径范围，不能判断动作语义。

修正：路径校验、权限规则、handler guard、OS sandbox 分层使用。

### 坑 6：MCP 工具不做命名空间和授权

外部 server 接进来以后，工具数量和副作用都会扩大。

修正：保持 `mcp__server__tool` 命名，按 server 和 tool 精确授权。

### 坑 7：多 agent 场景里让子 agent 自己弹审批

子 agent 往往没有用户终端，也不一定有批准权。

修正：设计权限冒泡：请求者发起，Lead 或用户批准，结果回到原请求。

---

## 十六、动手实验

本节实验对应实验手册编号：AHE-004、AHE-016。

### 实验 1：观察 s03 如何拒绝硬危险命令

**输入**：运行 s03，在临时目录里输入：

```text
Run rm -rf /.
```

**预期观察**：`check_deny_list()` 命中硬拒绝，handler 不执行，`tool_result` 返回 `Permission denied.`

**验证标准**：确认命令没有执行，并记录拒绝结果是否进入下一轮模型上下文。

### 实验 2：观察 ask 如何介入写文件

**输入**：让 agent 修改一个工作区外路径：

```text
Write hello to ../outside.txt.
```

**预期观察**：写出工作区会命中规则并触发询问。

**验证标准**：选择 deny 后，文件不应被写入；模型应看到拒绝结果。

### 实验 3：把 `ask_user()` 改成默认拒绝

**输入**：临时把 `ask_user()` 改成不读取 input，直接返回 `deny`。

**预期观察**：所有需要人工确认的动作都会被拒绝。

**验证标准**：说明这相当于什么权限模式，以及它和 hard deny 有什么不同。

### 实验 4：给 `git push` 加一条 deny 规则

**输入**：在 s03 的 `PERMISSION_RULES` 里增加对 `git push` 的拒绝或询问。

**预期观察**：模型即使提出 `bash("git push ...")`，也不能直接执行。

**验证标准**：解释为什么这应该是 runtime 规则，而不是写进 prompt。

### 实验 5：观察 worktree 删除保护

**输入**：阅读 s18 的 `remove_worktree()`，在临时仓库里创建有改动的 worktree，再尝试删除。

**预期观察**：默认拒绝删除有未提交改动的 worktree，除非显式 `discard_changes=true`。

**验证标准**：说明这属于 allow/deny 规则，还是空间隔离和生命周期保护。

### 实验 6：给 MCP 工具做风险分级

**输入**：阅读 s19 的 mock MCP server，把 `docs.search` 和 `deploy.trigger` 分成只读和破坏性两类。

**预期观察**：两者都进入工具池，但风险完全不同。

**验证标准**：写出一组规则：允许 `mcp__docs__*`，询问或拒绝 `mcp__deploy__trigger`。

---

## 自测问题

1. 为什么权限检查应该放在 tool_use 之后、handler 之前？
2. Prompt 约束和 runtime 权限有什么本质区别？
3. `safe_path()` 能解决哪些问题，不能解决哪些问题？
4. 为什么 deny 规则应该优先于 allow 规则？
5. 被拒绝的工具调用为什么也应该回填成 `tool_result`？
6. 什么时候应该 `ask`，什么时候应该直接 `deny`？
7. 多 agent 场景下，为什么权限可能需要冒泡到 Lead？
8. worktree 隔离和权限规则分别控制什么？
9. MCP 工具为什么必须带 server 命名空间？
10. 如果一个团队想允许 agent 自动改代码，但禁止自动 push，你会怎么设计规则和模式？

---

## 参考资料

### 官方主参考

- [Claude Code Permissions](https://code.claude.com/docs/en/permissions)
- [Claude Code Hooks Reference](https://code.claude.com/docs/en/hooks)
- [Claude Agent SDK MCP](https://code.claude.com/docs/en/agent-sdk/mcp)

### 社区教学项目

- [shareAI-lab/learn-claude-code](https://github.com/shareAI-lab/learn-claude-code)
- [s03_permission](https://github.com/shareAI-lab/learn-claude-code/tree/main/s03_permission)
- [s15_agent_teams](https://github.com/shareAI-lab/learn-claude-code/tree/main/s15_agent_teams)
- [s18_worktree_isolation](https://github.com/shareAI-lab/learn-claude-code/tree/main/s18_worktree_isolation)
- [s19_mcp_plugin](https://github.com/shareAI-lab/learn-claude-code/tree/main/s19_mcp_plugin)
