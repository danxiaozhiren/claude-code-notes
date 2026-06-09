# 上下文工程：Todo、Skill、Compact、Memory 怎么协同

> 社区项目核查日期：2026-06-08
> 官方文档核查日期：2026-06-08
> 作者：wt
> 适合读者：个人开发者、团队技术负责人、想理解 Agent Harness 上下文管理的工程师
> 本文定位：`learn-claude-code` 学习系列第 5 篇，承接第 4 篇 Hooks，解释 Todo、Skill、Compact、Memory、System Prompt 这些机制如何一起管理 agent 的工作上下文。
> 实践建议：本文的实验会生成 `.memory/`、`.transcripts/`、`.task_outputs/` 等目录，建议在临时目录运行教学代码。

---

## 先把边界说清楚

本文基于社区教学项目 `shareAI-lab/learn-claude-code`，不代表 Anthropic Claude Code 官方内部实现。

本文讨论的是 Claude Code-like Agent Harness 的通用上下文工程机制，不是在还原 Claude Code 源码。文中使用四类标签：

| 标签 | 含义 |
| --- | --- |
| 【官方事实】 | 来自 Claude Code 官方文档或官方公开材料 |
| 【社区教学实现】 | 来自 `shareAI-lab/learn-claude-code` 的 README 和 `code.py` |
| 【作者推导】 | 基于教学代码、官方边界和工程经验形成的解释 |
| 【动手观察】 | 读者可以本地复现的实验现象 |

特别说明：`learn-claude-code` 的部分 README 会讨论“真实 Claude Code”的实现细节。本文不把这些内容当成官方事实，只把它们当作社区作者的学习笔记和教学推导。涉及官方能力时，只以 Claude Code 官方文档为准。

前四篇已经把行动链路搭起来：

```text
Agent Loop
  -> Tool Use
  -> Permission
  -> Hooks
```

这一篇进入另一个核心问题：

> 一个 agent 跑久了，怎么决定哪些信息留在上下文里，哪些按需加载，哪些压缩掉，哪些长期记住？

---

## 一、核心认知：上下文不是越多越好，而是要按生命周期管理

很多人理解上下文工程时，会把它简化成一句话：

```text
把更多资料塞给模型。
```

这是最容易出问题的做法。

Agent Harness 面对的不是“资料不够”，而是“资料生命周期不同”。

| 信息类型 | 生命周期 | 适合机制 |
| --- | --- | --- |
| 当前任务步骤 | 当前会话、当前目标 | Todo |
| 领域操作规程 | 任务需要时加载 | Skill |
| 工具结果和对话历史 | 活跃上下文内，超限要压缩 | Compact |
| 用户偏好、项目背景、长期事实 | 跨压缩、跨会话 | Memory |
| 工具、工作区、模式、可用知识索引 | 每轮系统输入 | System Prompt assembly |

【作者推导】上下文工程的核心不是“多放”，而是回答四个问题：

```text
这条信息什么时候需要？
需要完整内容还是索引？
过期后能不能丢？
丢了以后能不能恢复？
```

这也是本文标题里五个机制的分工：

```text
Todo: 把当前目标拆成可追踪状态
Skill: 把领域知识按需加载
Compact: 把活跃上下文压回预算内
Memory: 把长期重要信息放到上下文之外
System Prompt: 按当前真实状态组装模型入口
```

---

## 二、官方事实：Claude Code 公开文档里的上下文边界

先看官方公开边界，再看社区教学实现。

【官方事实】Claude Code 的 Context window 文档说明，上下文窗口包含系统提示、对话历史、文件内容、工具输出、部分隐藏上下文等；`/context` 可以查看当前使用情况；上下文接近上限时会自动 compact，也可以手动运行 `/compact`。

【官方事实】同一文档还说明，Claude Code 在用户输入前会先加载多类上下文：当前对话历史、长期记忆、当前工作目录、项目级 `CLAUDE.md`、启用的 MCP 工具名、已安装 skill 的名称和描述；当模型使用某个 skill 时，该 skill 的完整内容会被加载进上下文。

【官方事实】Claude Code Skills 文档说明，Skill 是目录加 `SKILL.md`，带 YAML frontmatter；启动时只加载 skill 的名称和描述，完整内容在 Claude 判断需要该 skill 时才加载。Skills 文档还说明，skill 可以包含渐进式披露的附加文件、脚本和资源。

【官方事实】Claude Code Memory 文档说明，Claude Code 支持项目、用户、本地、企业等不同范围的记忆文件；项目记忆常见位置是 `./CLAUDE.md`，用户记忆是 `~/.claude/CLAUDE.md`；导入语法可以把其他文件并入记忆。文档也说明自动 memory 会在任务过程中保存用户偏好和事实，存放在用户目录下并在会话开始时加载。

【官方事实】Tools reference 目前把 `TodoWrite` 标为 disabled by default，并说明偏好使用 `TaskCreate`、`TaskUpdate`、`TaskComplete` 等任务工具。这一点和 `learn-claude-code` 的 s05 教学章不同：s05 用 `todo_write` 讲最小规划机制，本文按社区教学实现分析它的工程含义，不把它写成 Claude Code 当前官方工具状态。

这些官方事实给出一个重要启发：

| 机制 | 官方公开边界给出的启发 |
| --- | --- |
| Skill | 只把目录放进常驻上下文，正文按需加载 |
| Memory | 长期信息不要只靠当前会话 history |
| Compact | 上下文窗口是有限预算，需要自动和手动压缩 |
| Todo / Task | 多步骤工作需要结构化进度，而不是只靠模型“记得” |
| System Prompt | 不同上下文来源要被组织进稳定入口 |

---

## 三、五个机制分别解决什么问题

如果把上下文工程看成一个整体，五个机制不是平级堆叠，而是处理不同时间尺度。

| 机制 | 处理的问题 | 时间尺度 | 典型位置 |
| --- | --- | --- | --- |
| Todo | 当前任务接下来做什么 | 当前会话内 | tool_result / 状态展示 |
| Skill | 当前任务需要什么方法和知识 | 需要时加载 | tool_result / 附加文件 |
| Compact | 活跃上下文快满了怎么办 | 当前会话内 | messages 预处理 / 摘要 |
| Memory | 用户和项目长期事实怎么保留 | 跨会话 | 文件系统 + 索引 + 按需注入 |
| System Prompt | 当前运行状态怎么进入模型 | 每轮入口 | system prompt sections |

【作者推导】这五个机制如果混用，会出现典型错误：

| 错误做法 | 为什么错 |
| --- | --- |
| 把所有团队规范塞进 system prompt | 无关任务也付 token，噪声大 |
| 把当前步骤写进 Memory | 短期状态污染长期记忆 |
| 把用户长期偏好只写进 Todo | 会话结束就丢 |
| 只做 Compact 不做 Memory | 压缩后细节和偏好可能丢失 |
| 只做 Memory 不做 Compact | 当前上下文仍然会爆 |
| System Prompt 每轮全量重拼 | 破坏缓存，增加成本和不稳定性 |

所以第 5 篇不是讲“五个功能”，而是讲“五类上下文材料怎么协同”。

---

## 四、Todo：当前任务状态，不是执行能力

【社区教学实现】s05 新增 `todo_write` 工具。它接收一个列表，每项有 `content` 和 `status`，状态是：

```text
pending
in_progress
completed
```

最小结构可以写成：

```python
def run_todo_write(todos):
    todos, error = normalize_todos(todos)
    if error:
        return error
    CURRENT_TODOS = todos
    return f"Updated {len(CURRENT_TODOS)} tasks"
```

s05 还加入一个 reminder：如果连续几轮没有更新 todo，就向 messages 注入提醒。

```text
if rounds_since_todo >= 3:
    messages.append("<reminder>Update your todos.</reminder>")
```

【作者推导】Todo 的关键价值不是“多了一个工具”，因为它不读文件、不写文件、不执行命令。它给 agent 增加的是当前任务的可见状态。

没有 Todo 时，长任务容易变成：

```text
做了第 1 步
临时发现一个问题
修了另一个问题
忘了第 4-8 步
最后给出一个看似完成的总结
```

有 Todo 后，模型每隔几轮会重新看到：

```text
哪些已经完成？
哪个正在进行？
哪些还没开始？
```

这不是长期记忆，而是当前工作现场。

| Todo 应该保存 | Todo 不应该保存 |
| --- | --- |
| 当前任务步骤 | 用户长期偏好 |
| 本轮进度 | 团队规范全文 |
| 待验证项 | 大量文件内容 |
| 阻塞项 | 历史会话经验 |

---

## 五、Skill：知识按需加载，而不是全塞 prompt

【社区教学实现】s07 的 Skill Loading 采用两层结构：

```text
启动时:
  扫描 skills/
  把 skill name + description 放进 system prompt

运行时:
  模型需要某个 skill
  调用 load_skill(name)
  完整 SKILL.md 作为 tool_result 回填
```

代码结构可以压缩成：

```python
def scan_skills():
    for d in skills_dir:
        meta, body = parse_frontmatter(d / "SKILL.md")
        registry[name] = {
            "description": meta["description"],
            "content": raw_skill_text,
        }

def build_system():
    return "Skills available:\n" + list_names_and_descriptions()

def load_skill(name):
    return registry[name]["content"]
```

【官方事实】Claude Code Skills 文档也公开了类似思想：启动时只加载 skill 名称和描述，完整 skill 在需要时加载；skill 还可以通过附加文件、脚本和资源做渐进式披露。

【作者推导】Skill 解决的是“方法知识”的上下文问题。

例如：

```text
写 React 组件
做代码审查
生成 PR 描述
迁移测试夹具
格式化某类文档
```

这些不是一个工具动作，而是一套流程和判断标准。它们也不应该永久塞进 system prompt，因为不是每个任务都需要。

| 方案 | 成本 | 风险 |
| --- | --- | --- |
| 全部 skill 正文塞进 system prompt | 每轮都付 token | 噪声大，干扰当前任务 |
| 只列 skill 名称和描述 | 常驻成本低 | 模型需要会主动加载 |
| 用 `load_skill` 加正文 | 按需付成本 | 需要 skill 描述写得清楚 |
| skill 内再引用附加文件 | 渐进式披露 | 需要文件边界和路径规则 |

Skill 的好描述非常重要。描述太泛，模型不知道什么时候加载；描述太窄，模型会错过它。

---

## 六、Compact：上下文预算管理，不是“随便摘要一下”

【社区教学实现】s08 把 compact 做成四层管线：

| 层级 | 机制 | 目的 | 成本 |
| --- | --- | --- | --- |
| L1 | `snip_compact` | 裁掉中间旧消息，保留开头和尾部 | 0 API |
| L2 | `micro_compact` | 把旧 tool_result 替换成占位符 | 0 API |
| L3 | `tool_result_budget` | 大输出落盘，只留路径和预览 | 0 API |
| L4 | `compact_history` | 用 LLM 生成摘要 | 1 次 API |
| 应急 | `reactive_compact` | API 报 prompt too long 后兜底 | 1 次 API |

s08 的关键细节是：压缩不能破坏 tool_use 和 tool_result 的配对。

```text
assistant(tool_use id=abc)
user(tool_result tool_use_id=abc)
```

如果裁剪时留下孤立的 `tool_result`，模型会看到一个没有请求来源的结果；如果留下孤立的 `tool_use`，协议也断了。

【作者推导】Compact 解决的是活跃上下文预算，不是长期记忆。

它的目标是：

```text
让当前任务继续跑下去
```

不是：

```text
永久保存所有细节
```

这决定了 compact 的取舍：

| 可以压缩 | 需要保留或可恢复 |
| --- | --- |
| 很久以前的完整工具输出 | 当前目标 |
| 已经用过的大日志 | 关键决策 |
| 旧文件内容 | 已修改文件 |
| 重复命令输出 | 剩余工作 |
| 冗余对话 | 用户明确约束 |

所以 s08 的摘要 prompt 要求保留当前目标、关键发现、文件读改情况、剩余工作、用户约束。这不是写读后感，而是给下一轮 agent 续接任务。

---

## 七、Memory：跨压缩、跨会话的长期知识

s08 的 compact 能把当前会话压回预算内，但它有损。

如果用户说：

```text
以后写这个项目的文章都要明确区分官方事实和作者推导。
```

这不应该只留在当前 messages，也不应该只出现在某次 compact summary 里。它是跨会话仍然有用的偏好。

【社区教学实现】s09 用 `.memory/` 目录实现 Memory：

```text
.memory/
  MEMORY.md
  writing-style.md
  project-boundary.md
```

每个记忆文件带 frontmatter：

```markdown
---
name: writing style
description: Chinese technical articles must separate source types
type: preference
---

...
```

索引文件 `MEMORY.md` 只保存名称、链接和描述。加载时分两步：

```text
1. MEMORY.md 索引进入 system prompt
2. 根据当前对话选择相关 memory 文件，把正文按需注入 user turn
```

s09 的核心函数可以概括为：

| 函数 | 作用 |
| --- | --- |
| `write_memory_file()` | 写单条记忆文件并重建索引 |
| `_rebuild_index()` | 生成 `MEMORY.md` 索引 |
| `select_relevant_memories()` | 根据当前对话从索引中选择相关文件 |
| `load_memories()` | 读取被选中的记忆正文并注入 |
| `extract_memories()` | 本轮结束后从对话中提取新记忆 |
| `consolidate_memories()` | 低频合并、去重、淘汰过时记忆 |

【官方事实】Claude Code Memory 文档公开了项目、用户、本地、企业等记忆范围，也说明自动 memory 可以保存用户偏好和事实并在会话开始时加载。

【作者推导】Memory 和 Compact 的关系是：

```text
Compact: 让当前会话继续
Memory: 让未来会话也知道
```

Memory 应该保存“以后还会用”的信息，而不是所有聊天历史。

| 适合 Memory | 不适合 Memory |
| --- | --- |
| 用户稳定偏好 | 临时命令输出 |
| 项目长期边界 | 本轮中间推理 |
| 常见排查路径 | 一次性 todo |
| 写作纪律 | 过期事实 |
| 反复出现的反馈 | 大文件全文 |

---

## 八、System Prompt：运行时组装，不是硬编码大段字符串

Todo、Skill、Compact、Memory 都会改变当前上下文状态。

如果 system prompt 仍然是一大段硬编码字符串，后面会越来越难维护：

```text
工具变了，要改 prompt
skills 变了，要改 prompt
memory 有无变化，要改 prompt
工作区变了，要改 prompt
模式变了，要改 prompt
```

【社区教学实现】s10 把 system prompt 拆成 sections：

```python
PROMPT_SECTIONS = {
    "identity": "...",
    "tools": "...",
    "workspace": "...",
    "memory": "...",
}
```

然后根据真实 context 拼接：

```python
def assemble_system_prompt(context):
    sections = [identity, tools, workspace]
    if context["memories"]:
        sections.append(memory_section)
    return "\n\n".join(sections)
```

再用稳定序列化做缓存：

```python
key = json.dumps(context, sort_keys=True)
if key == last_context_key:
    return last_prompt
```

【作者推导】System Prompt assembly 是上下文工程的总装层。

它不负责保存所有信息，而是负责决定：

```text
这一轮模型启动时，应该看到哪些稳定入口？
```

| section | 何时常驻 | 何时按需 |
| --- | --- | --- |
| identity | 通常常驻 | 很少变化 |
| tools | 工具池变化时更新 | MCP 动态连接时重组 |
| workspace | 工作区变化时更新 | worktree 切换时更新 |
| memory index | 可常驻索引 | 正文按需注入 |
| skill catalog | 常驻名称描述 | 正文按需加载 |
| project rules | 项目内常驻 | 大文档按需引用 |

这也是为什么 s10 不应该被看成“prompt 写法技巧”。它是把前面所有上下文来源变成可组合 section。

---

## 九、五个机制如何协同：一条长任务轨迹

假设用户说：

```text
重构认证模块，遵守本仓库写作/编码规范，跑完必要检查，并记住这次发现的约束。
```

一个比较健康的 harness 轨迹应该是：

```text
1. System Prompt
   - 注入工具列表、工作区、memory index、skill catalog

2. Todo
   - 先列出阅读、定位、修改、测试、总结步骤

3. Skill
   - 如果任务涉及代码审查或项目规范，加载对应 skill 正文

4. Tool Use
   - 读文件、搜索、编辑、运行检查

5. Compact
   - 工具结果过多时压缩旧结果
   - 大输出落盘
   - 接近上限时摘要当前工作

6. Memory
   - 任务结束后提取稳定偏好、项目约束、常见路径
   - 下次会话通过索引和相关记忆加载回来

7. System Prompt 下一轮重组
   - 根据新的 memory index、工具状态、工作区状态更新入口
```

【作者推导】这条轨迹里，每个机制只做自己的事：

| 机制 | 不越界的表现 |
| --- | --- |
| Todo | 不保存长期偏好，只追踪当前任务 |
| Skill | 不常驻全文，只按需加载 |
| Compact | 不假装永久记忆，只保留续接摘要 |
| Memory | 不记录所有历史，只保存可复用事实 |
| System Prompt | 不塞满所有正文，只组装当前入口 |

如果某个机制越界，系统就会变形。

---

## 十、深水区一：索引常驻，正文按需

Skill 和 Memory 都有一个共同设计：

```text
索引常驻
正文按需
```

Skill 的索引是：

```text
skill name + description
```

Memory 的索引是：

```text
memory name + description + link
```

正文都不应该无脑常驻。

【作者推导】这是上下文工程里最重要的 token 经济学：

| 设计 | 成本 | 效果 |
| --- | --- | --- |
| 正文全常驻 | 高 | 模型每轮都看到大量无关信息 |
| 只有索引常驻 | 低 | 模型知道“有什么可用” |
| 正文按需加载 | 中 | 当前任务需要时才付成本 |
| 附加资源再按需读取 | 更可控 | 大文件、脚本、模板不会污染上下文 |

这也解释了为什么 skill 描述和 memory 描述要写好。索引描述是模型决定是否加载正文的入口。

---

## 十一、深水区二：活跃上下文和持久上下文要分开

活跃上下文是当前 messages 里模型马上能看到的内容。

持久上下文是文件系统、记忆库、transcript、skill 目录、项目文档等外部存储。

两者的关系应该是：

```text
持久上下文
  -> 通过索引、筛选、工具调用、prompt section
  -> 进入活跃上下文
```

而不是：

```text
持久上下文全部塞进活跃上下文
```

【作者推导】这能避免两个极端：

| 极端 | 问题 |
| --- | --- |
| 什么都放活跃上下文 | 成本高、噪声大、容易超限 |
| 什么都放外部文件 | 模型不知道存在，无法主动使用 |

好的 harness 应该让模型知道“外部有什么”，并提供工具或机制按需取回。

---

## 十二、深水区三：Compact 和 Memory 的时机不同

Compact 通常发生在：

```text
每轮 LLM 前
上下文接近阈值
模型主动请求 compact
API 报 prompt too long
```

Memory 通常发生在：

```text
用户表达稳定偏好
任务结束后提取
定期整理去重
下次会话开始加载
```

【社区教学实现】s09 在 loop 里先保存压缩前快照 `pre_compress`，任务停止时从这个快照里提取记忆。

这很关键。

如果从 compact 后的摘要里提取 memory，就可能把已经丢失或泛化的细节写成长期记忆。

【作者推导】可以这样理解：

```text
Compact 从丰富历史中提取“当前续接需要”
Memory 从丰富历史中提取“未来还会复用”
```

两者都可能用 LLM 摘要，但目标完全不同。

---

## 十三、深水区四：不要让 Memory 变成垃圾场

Memory 很容易滥用。

如果每轮都“记住”一堆细节，记忆库会膨胀，后果是：

```text
索引越来越长
选择越来越慢
相关性越来越差
旧事实和新事实冲突
模型开始被过期偏好误导
```

所以 Memory 必须有写入标准：

| 应该写入 | 不应该写入 |
| --- | --- |
| 用户明确要求记住 | 一次性闲聊 |
| 反复出现的偏好 | 临时 debug 输出 |
| 项目长期边界 | 当前 todo 进度 |
| 常用命令和入口 | 随手读到的大段文件 |
| 纠错后的规则 | 未验证的模型猜测 |

s09 的 `consolidate_memories()` 说明了另一个关键点：Memory 不只要写入，还要整理。

长期记忆如果没有去重、合并、淘汰，就会从资产变成负担。

---

## 十四、深水区五：System Prompt 要稳定，但不能僵硬

System prompt 处在一个矛盾位置：

```text
太稳定：无法反映当前工具、memory、workspace 状态
太动态：破坏缓存，增加噪声，难以复现
```

s10 的解法是 section 化：

```text
稳定 section: identity, core rules, tool policy
动态 section: memory index, workspace, enabled tools, MCP instructions
```

【作者推导】section 化有三个收益：

| 收益 | 说明 |
| --- | --- |
| 可维护 | 新增 memory section 不影响 identity |
| 可缓存 | 稳定部分可以尽量保持不变 |
| 可解释 | 出问题时能知道是哪一段上下文影响模型 |

但 dynamic section 要克制。不要把每轮所有临时状态都塞进 system prompt。很多临时状态更适合 user reminder、tool_result 或 hook 注入。

---

## 十五、机制解释：上下文工程为什么能支撑长任务

把所有机制合起来，一条长任务的上下文流可以写成：

```text
外部持久材料
  - skills/
  - .memory/
  - project docs
  - transcripts

索引层
  - skill catalog
  - MEMORY.md
  - tool list

活跃上下文
  - system prompt sections
  - recent messages
  - current todo state
  - relevant skill content
  - relevant memory content
  - recent tool results

预算治理
  - micro compact
  - tool result budget
  - auto compact
  - reactive compact

长期回流
  - memory extraction
  - consolidation
  - next-session loading
```

这个结构能支撑长任务，是因为它把上下文分成了不同层：

| 层 | 作用 |
| --- | --- |
| 常驻索引 | 让模型知道有哪些外部材料 |
| 按需正文 | 当前任务需要时才加载 |
| 当前状态 | 让模型知道下一步该做什么 |
| 压缩摘要 | 让当前会话继续 |
| 长期记忆 | 让未来会话继承稳定知识 |
| 组装入口 | 每轮把真实状态带进模型 |

【作者推导】这比“把所有东西塞给模型”更难，但更可持续。

---

## 十六、最小代码理解：抓住 10 个代码点

读 s05、s07、s08、s09、s10 时，先抓住这些点。

| 代码点 | 来自章节 | 看什么 | 为什么重要 |
| --- | --- | --- | --- |
| `run_todo_write()` | s05 | 当前任务状态如何写入 | Todo 是计划状态，不是执行能力 |
| `rounds_since_todo` | s05 | reminder 何时注入 | harness 可提示模型维护计划 |
| `_scan_skills()` | s07 | skill catalog 如何构建 | 索引常驻的来源 |
| `load_skill()` | s07 | 正文如何按需进入 tool_result | 知识按需加载 |
| `snip_compact()` | s08 | 裁剪历史时保留边界 | 不能破坏 tool_use/result |
| `micro_compact()` | s08 | 旧工具结果占位 | 降低上下文负担 |
| `tool_result_budget()` | s08 | 大输出落盘 | 活跃上下文只留路径和预览 |
| `compact_history()` | s08 | LLM 摘要续接任务 | 压缩当前会话 |
| `load_memories()` / `extract_memories()` | s09 | 记忆加载和提取 | 跨压缩、跨会话 |
| `assemble_system_prompt()` | s10 | 根据真实 context 拼 section | 上下文总装入口 |

最小结构图：

```text
每轮开始:
  context = update_context()
  system = assemble_system_prompt(context)
  load relevant memories
  compact messages if needed

模型行动:
  todo_write 维护当前计划
  load_skill 按需加载知识
  tools 产生 observation

每轮结束:
  compact 保持会话可继续
  extract memory 保存长期事实
  system prompt 下轮按新状态重组
```

---

## 十七、常见坑和修正

### 坑 1：把 Todo 当成 Memory

当前任务步骤不应该长期保存。

修正：Todo 管当前目标，Memory 管跨会话仍有价值的信息。

### 坑 2：把 Skill 正文全塞 system prompt

无关任务也会付 token，还会稀释注意力。

修正：system prompt 只放 skill catalog，正文通过 `load_skill` 按需加载。

### 坑 3：只做 Compact，不做 Memory

Compact 会有损，用户长期偏好可能被摘要弱化。

修正：把稳定偏好、项目边界、常用入口提取成 Memory。

### 坑 4：Memory 无限制写入

记忆库膨胀会降低相关性。

修正：只写稳定、可复用、已确认的信息；定期 consolidation。

### 坑 5：压缩时破坏 tool_use / tool_result 配对

协议链断了，模型会看到无法解释的结果。

修正：压缩时按消息组处理，不留下孤立工具请求或孤立结果。

### 坑 6：System Prompt 过度动态

每轮都重写大段 system prompt 会增加成本和不稳定性。

修正：拆 section，稳定部分保持稳定，动态部分按真实状态加载。

### 坑 7：把大输出直接塞回 messages

一次日志或文件读取就可能打爆上下文。

修正：用 tool result budget 落盘，只保留路径、摘要和预览。

---

## 十八、动手实验

### 实验 1：观察 Todo 对长任务的影响

**输入**：运行 s05，让 agent 完成一个 4 步以上的小重构。

**预期观察**：第一次或早期工具调用应出现 `todo_write`，后续状态从 `pending` 到 `in_progress` 到 `completed`。

**验证标准**：记录 agent 是否遗漏步骤，以及 reminder 是否触发。

### 实验 2：比较 Skill catalog 和正文加载

**输入**：运行 s07，先观察 system prompt 里的 skill catalog，再让模型加载一个 skill。

**预期观察**：启动时只有名称和描述；调用 `load_skill` 后完整 `SKILL.md` 才进入 tool_result。

**验证标准**：说明为什么这比全量 system prompt 更省上下文。

### 实验 3：制造大工具输出

**输入**：运行 s08，让 agent 读取一个很大的文件或生成大量输出。

**预期观察**：大结果被落盘，messages 中只保留 `<persisted-output>` 和预览。

**验证标准**：找到落盘路径，并解释模型如何重新读取完整内容。

### 实验 4：触发 compact

**输入**：在 s08 中反复读多个文件或对话多轮，观察 `[auto compact]`。

**预期观察**：旧 messages 被摘要替换，transcript 被保存。

**验证标准**：摘要是否保留当前目标、关键发现、文件读改情况、剩余工作和用户约束。

### 实验 5：写入并加载 Memory

**输入**：运行 s09，多轮告诉 agent 一个稳定偏好，例如：

```text
Remember that I prefer Chinese technical articles to separate official facts, community implementation, author inference, and hands-on observation.
```

**预期观察**：`.memory/` 下生成 `.md` 文件，`MEMORY.md` 索引更新；后续相关任务会加载该记忆。

**验证标准**：确认记忆没有只留在当前 messages 中。

### 实验 6：观察 System Prompt section 变化

**输入**：运行 s10，先无 `.memory/MEMORY.md`，再创建该文件。

**预期观察**：memory section 只在真实文件存在且有内容后加载。

**验证标准**：说明它是根据真实状态，而不是根据用户关键词组装 prompt。

---

## 自测问题

1. Todo、Skill、Compact、Memory 分别处理哪种时间尺度的信息？
2. 为什么 Skill 应该“索引常驻，正文按需”？
3. Compact 和 Memory 都可能用摘要，它们目标有什么不同？
4. 为什么 Memory 不应该保存当前 todo 进度？
5. `tool_result_budget()` 为什么要把大输出落盘，而不是直接删除？
6. 压缩时为什么不能拆散 tool_use 和 tool_result？
7. System Prompt section 化解决了什么维护问题？
8. 什么信息应该放 system prompt，什么信息应该放 user reminder 或 tool_result？
9. 如果 agent 总是忘记验证，你会用 Todo、Hook、Memory 还是 Skill 解决？为什么？
10. 如果一个项目有 20 个规范文档，你会如何设计 skill catalog、load_skill 和 memory index？

---

## 参考资料

### 官方主参考

- [Claude Code Context Window](https://code.claude.com/docs/en/context-window)
- [Claude Code Skills](https://code.claude.com/docs/en/skills)
- [Claude Code Memory](https://code.claude.com/docs/en/memory)
- [Claude Code Tools Reference](https://code.claude.com/docs/en/tools-reference)

### 社区教学项目

- [shareAI-lab/learn-claude-code](https://github.com/shareAI-lab/learn-claude-code)
- [s05_todo_write](https://github.com/shareAI-lab/learn-claude-code/tree/main/s05_todo_write)
- [s07_skill_loading](https://github.com/shareAI-lab/learn-claude-code/tree/main/s07_skill_loading)
- [s08_context_compact](https://github.com/shareAI-lab/learn-claude-code/tree/main/s08_context_compact)
- [s09_memory](https://github.com/shareAI-lab/learn-claude-code/tree/main/s09_memory)
- [s10_system_prompt](https://github.com/shareAI-lab/learn-claude-code/tree/main/s10_system_prompt)
