# 工具系统：Claude Code 为什么能“自己动手”

> 文档核查日期：2026-05-18  
> 作者：数据室 伍涛。  
> 写作初衷：在斌总和葛处的鼓励与倡导下，围绕 Claude Code 做一次面向底层原理的系统学习，把学习过程中的理解、积累和实践经验沉淀下来，并以文章形式分享给更多同事和读者。  
> 适合读者：个人开发者、团队负责人、平台开发者  
> 本文定位：Claude Code 系列第 3 篇，聚焦工具系统如何把模型从“回答问题”变成“执行任务”。  
> 实践建议：本文末尾提供动手练习，建议读者用真实任务观察工具调用如何改变后续判断。

---

## 先把边界说清楚

第 1 篇讲的是总模型：Claude Code 不是聊天工具，而是 agentic harness。

第 2 篇讲的是静态 context 装载：CLAUDE.md、Auto memory、rules、skills、MCP tool names 等内容如何在会话开始或特定时机进入 context。

这篇只讲一个问题：

> Claude Code 为什么能从“回答你”变成“自己动手做”？

本文关注的是 context 的动态填充：

- Claude 为什么会先搜索、再读取、再编辑、再验证
- Read / Edit / Bash / Grep / Glob / WebFetch 等工具分别扮演什么角色
- 工具输出如何进入 context，并影响下一步判断
- 为什么大日志、大文件、无关命令输出会污染 context
- code intelligence 和普通文本搜索有什么区别
- 工具权限为什么不是附属功能，而是行动边界

不展开的内容：

- CLAUDE.md、Auto memory、rules、skills 的装载机制：见第 2 篇
- permissions、sandbox、prompt injection 的系统安全设计：放到第 4 篇
- MCP、hooks、subagents 的组合架构：放到后续文章

本文只在容易混淆的地方标注来源类型：

- 【官方事实】：来自 Claude Code 官方文档或官方仓库。
- 【作者推导】：基于官方事实整理出的工程判断。
- 【实践练习】：读者可以在自己的环境中复现观察的部分。

---

## 一、核心认知：工具是 agentic loop 的行动层

如果 Claude 只能输出文字，它就是一个会解释代码的模型。

有了工具，Claude Code 才能进入工程现场：

```text
读文件
  -> 搜索线索
  -> 运行测试
  -> 看错误输出
  -> 修改代码
  -> 再运行测试
  -> 根据新结果继续调整
```

【官方事实】官方 How Claude Code works 文档说，工具让 Claude Code 具备 agentic 能力。没有工具时，Claude 只能用文字回应；有工具后，它可以读取代码、编辑文件、运行命令、搜索 Web、和外部服务交互。每次工具使用都会返回信息，这些信息会反馈回 loop，影响 Claude 的下一步决策。

【官方事实】官方文档把 Claude Code 的内置工具能力大致分成五类：

| 类别 | 能做什么 | 典型工具 |
| --- | --- | --- |
| File operations | 读文件、改代码、创建文件、重组文件 | Read、Edit、Write |
| Search | 按文件名找文件、按内容搜模式、探索代码库 | Glob、Grep |
| Execution | 运行 shell 命令、启动服务、跑测试、使用 git | Bash、PowerShell、Monitor |
| Web | 搜索网页、抓取文档、查询错误信息 | WebSearch、WebFetch |
| Code intelligence | 类型错误、跳转定义、查引用、调用层级 | LSP |

【作者推导】这五类工具对应的是五种工程动作：

- 看见：Read / Grep / Glob / LSP
- 改变：Edit / Write / NotebookEdit
- 验证：Bash / PowerShell / LSP
- 外联：WebFetch / WebSearch / MCP
- 隔离和组织：Agent / Task / Worktree 等编排工具

其中 Agent / Task / Worktree 这类能力不是本文重点。它们用于把调研、审查、验证或并行开发拆到独立上下文或独立工作区，是多 agent 架构的基础，会在第 6 篇展开。

所以工具不是 Claude Code 的“插件列表”，而是它进入 agentic loop 的行动层。

---

## 二、工具输出如何变成下一步判断

一个常见误解是：Claude Code 先在脑子里想好完整方案，然后按顺序执行。

真实情况更接近：

```text
先做一个能降低不确定性的动作，
看动作结果，
再决定下一步。
```

【官方事实】官方文档说，Claude 会根据提示和过程中学到的信息选择工具。比如你说“修复失败测试”时，它可能先运行测试，读取错误输出，搜索相关源文件，读取这些文件，编辑代码，再重新运行测试。

【作者推导】为什么不能在一开始把所有信息装进 context，再一次性给出完整方案？

因为语言模型的一次生成只能基于当前 context 做推理。它不能在生成前主动获得还没读取的文件、还没运行的测试、还没发生的命令输出。如果把整个仓库、所有日志、所有依赖都提前塞进 context，会遇到两个问题：一是 context window 有限，重要信息会被挤压；二是多数信息和当前任务无关，反而增加噪音。

工具调用就是折中方案：先用当前 context 判断“最值得获取的下一块信息”，再通过工具拿到真实反馈，把反馈追加进 context，然后进入下一轮推理。

这背后的机制是：

```text
tool call
  -> tool result
  -> result enters context
  -> Claude sees result
  -> next decision changes
```

【官方事实】官方 context window 文档说明：Claude 工作时，每次文件读取都会增加 context；path-scoped rules 会随匹配文件自动加载；编辑后 PostToolUse hook 会触发。

【作者推导】这说明“工具反馈”和“context 填充”是同一条链路上的两个视角：

- 从行为上看：Claude Code 在根据反馈调整路线。
- 从 context 上看：工具结果进入当前会话，成为下一轮推理能看见的新材料。

例如：

```text
用户：测试失败了，帮我修。

Bash 输出：Expected 200, received 500
Claude 下一步：不再泛泛猜测，而是搜索 500 来源。

Grep 输出：错误集中在 session/refresh
Claude 下一步：读取 session/refresh 实现。

Read 输出：发现 token 过期分支没有返回值
Claude 下一步：编辑该分支。

Bash 输出：测试通过
Claude 下一步：总结改动和风险。
```

这里每一次工具结果都不是“附加日志”，而是下一步决策的证据。

---

## 三、常用工具各自解决什么问题

【官方事实】Tools reference 列出了 Claude Code 可用工具及权限要求。不同工具不是同一个“读写能力”的变体，而是解决不同粒度的问题。

### 3.1 Read：精确读取上下文

Read 用来读取文件内容。

【官方事实】Read 返回带行号的文件内容，并要求传入绝对路径。默认从文件开头读取；文件超过大小阈值时，会提示 Claude 用 `offset` 和 `limit` 分段读取。

【作者推导】Read 适合在 Claude 已经知道“要看哪个文件”时使用。它的价值不是搜索，而是把具体内容放进 context，让后续判断有依据。

适合：

- 读关键实现文件
- 读测试文件
- 读配置文件
- 分段读大文件

不适合：

- 在不知道位置时盲目读很多文件
- 把长日志整段塞进 context

### 3.2 Glob：先找文件

Glob 按文件名模式找文件。

【官方事实】Glob 支持标准 glob 语法，包括 `**` 递归匹配；结果按修改时间排序，最多返回 100 个文件。Glob 默认不遵循 `.gitignore`，这一点和 Grep 不同。

【作者推导】Glob 适合回答“可能有哪些文件相关”。它不告诉你文件内容，只帮 Claude 缩小候选范围。

典型用法：

```text
找所有 payment 相关测试文件
找 src 下的 route.ts
找最近修改的 markdown 文档
```

如果 Glob 返回太多，说明问题还不够具体，Claude 需要换更窄的 pattern。

### 3.3 Grep：再找线索

Grep 按内容搜索。

【官方事实】Grep 基于 ripgrep，使用 ripgrep 的正则语法。它可以返回匹配文件、匹配行或计数；可以按文件 glob 或语言类型收窄范围。默认情况下，Grep 会遵循 `.gitignore`。

【作者推导】Grep 适合回答“这个符号、错误、接口、文本在哪里出现”。它介于 Glob 和 Read 之间：

```text
Glob 找候选文件
Grep 找候选位置
Read 读取关键上下文
```

一个健康的工具链路经常是：

```text
Grep: 找到 useSession 出现在哪些文件
Read: 读取最相关的实现和测试
Edit: 修改最小范围
Bash: 跑相关测试
```

### 3.4 Edit / Write：改变文件

Edit 做定点修改，Write 创建或覆盖文件。

【官方事实】Edit 使用精确字符串替换，不是正则或模糊匹配。它要求 Claude 已经在当前会话中读取过目标文件，并且文件在读取后没有被外部改变。替换内容还要能精确匹配，并且默认需要唯一匹配。

【作者推导】这解释了为什么 Claude Code 有时会先 Read 再 Edit，也解释了为什么你在外部编辑器里改了文件后，它可能需要重新读取。

这个要求不是任意规则，而是精确字符串替换带来的必然约束。Edit 需要把 `old_string` 在当前文件里精确匹配出来，再替换成 `new_string`。如果 Claude 没有先读取文件，它就不知道当前文件的真实文本；如果外部程序在读取后改了文件，先前那份文本快照可能已经失效，`old_string` 可能不存在、重复出现，或者周边语义已经变化。

所以 read-before-edit 的本质是：先建立当前文件快照，再基于快照做局部替换。

理解这一点后，很多 Edit 失败都能预判：

- 目标文件没有在当前会话读过
- 你在编辑器里刚改过文件
- `old_string` 不够长，匹配到多处
- 文件格式化后，空格或换行已经变化

Edit 的约束是好事。它降低了“凭印象改文件”的风险。

适合：

- 小范围修改
- 保留周边结构
- 精确替换一段逻辑

Write 更适合：

- 新建文档
- 新建测试文件
- 生成完整配置文件

但覆盖已有文件风险更高，通常应更谨慎。

NotebookEdit 是用于编辑 Jupyter notebook 单元格的工具。本质上它也属于写入类工具，只是修改对象从普通文本文件变成 notebook cell；同样需要关注修改前后的 diff、执行结果和隐藏状态。

### 3.5 Bash / PowerShell：运行真实环境

Bash 和 PowerShell 让 Claude Code 运行命令。

【官方事实】Bash 命令以独立进程运行。`cd` 在主会话中可以延续到后续 Bash 命令，但前提是仍在项目目录或额外允许的工作目录内；环境变量不会跨命令持久化。Bash 默认超时时间为 2 分钟，命令输出默认限制为 30,000 字符。超过限制时，Claude Code 会把完整输出保存到 session 目录文件里，并只给 Claude 文件路径和开头预览。

【作者推导】Bash 是 Claude Code 从“看代码”进入“验证现实”的关键工具。测试结果、构建错误、lint 输出、git 状态，都比模型猜测更可靠。

但 Bash 也是风险最高的工具之一：

- 可能修改文件
- 可能启动服务
- 可能访问网络
- 可能触发部署脚本
- 可能产生巨大输出

所以 Bash 和 PowerShell 往往需要权限确认，也应该配合清晰的任务边界。

Monitor 可以理解成 Bash / PowerShell 的后续观察工具：Bash 负责启动命令或服务，Monitor 用来查看长时间运行进程的新输出。它适合观察 dev server、watch 测试、后台构建、长时间迁移脚本等持续运行任务。

它和 Bash 的组合方式通常是：

```text
Bash：启动 npm run dev 或测试 watch
Monitor：过一会儿查看新输出
Claude：根据新输出决定继续等待、修 bug、停止服务，还是读取日志文件
```

Monitor 不适合替代一次性命令。一次性命令直接用 Bash / PowerShell 就够了；Monitor 的价值在于持续进程不会一次性结束，Claude 需要后续观察它的输出变化。

### 3.6 WebFetch / WebSearch：连接外部文档

WebFetch 和 WebSearch 让 Claude Code 查外部资料。

【官方事实】WebFetch 会抓取指定 URL，并把网页转成 Markdown，再用一个小模型按提示抽取内容；Claude 收到的是处理后的结果，不一定是原始页面。大型页面会被截断，重复抓取有缓存，访问新域名通常会触发权限确认。

【作者推导】WebFetch 适合快速核对官方文档、错误信息和 API 行为，但它不是网页快照工具。对于关键事实，应该记录 URL 和核查日期；对于需要精确原文的位置，最好回到官方文档或仓库。

这也是本文和整个系列坚持“文档核查日期”的原因。

### 3.7 LSP：代码理解不是纯文本搜索

LSP 提供语言服务器能力。

【官方事实】LSP 可以在编辑后报告类型错误和警告，也可以用于跳转定义、查引用、查看类型信息、列出符号、找接口实现、追踪调用层级。它需要安装对应语言的 code intelligence plugin，并且语言服务器需要单独安装。

【作者推导】Grep 是文本搜索，LSP 是语言结构搜索。

更准确地说，Grep 工作在词法层：它只知道某段文本有没有出现。一个函数名叫 `handleAuth`，Grep 会找到所有包含这个字符串的位置，包括注释、字符串字面量、测试描述、另一个模块里恰好同名的函数。

LSP 工作在语义层：语言服务器会解析项目、作用域、类型、定义和引用关系。跳转定义不是“搜索同名文本”，而是基于当前符号在语言语义里的绑定关系找到定义点。

这在多态、接口实现、重载、同名函数、跨模块引用里尤其重要。Grep 的结果可能是线索，也可能是噪音；LSP 的结果更接近“这个符号在程序结构里的位置”。

两者不是替代关系：

| 问题 | 更适合 |
| --- | --- |
| 某个字符串在哪里出现 | Grep |
| 某个函数定义在哪里 | LSP |
| 某个接口有哪些实现 | LSP |
| 哪些文件名匹配模式 | Glob |
| 某段实现具体怎么写 | Read |
| 编辑后有没有类型错误 | LSP / Bash |

没有 LSP 时，Claude Code 仍然可以通过 Grep、Read、Bash 做大量工作；有 LSP 时，它能更准确地理解符号关系，减少误读和误改。

工程判断可以这样用：如果只是找错误码、配置键、日志文本、函数名的大致出现位置，Grep + Read 通常够用；如果任务涉及同名符号、接口实现、重载、继承、多态、跨模块调用链，误判成本就会明显升高，优先启用 LSP，或者至少用测试、类型检查和更小范围的 Read 来兜底。

---

## 四、大输出为什么会污染 context

工具输出进入 context 是 Claude Code 的优势，也是风险。

【官方事实】context window 文档说明：Claude 工作时，每次文件读取都会增加 context。官方还建议用 `/context` 查看当前 session 的实际 context 使用情况。

【官方事实】Bash 工具有输出长度限制：默认 30,000 字符。超过限制时，Claude Code 保存完整输出到 session 文件，并把文件路径和短预览给 Claude，Claude 需要时再读取或搜索完整文件。

【作者推导】这说明官方已经在工具层面处理“大输出不能无脑塞进 context”的问题。但对使用者来说，仍然要理解两类污染：

### 4.1 空间污染

无关内容占用 context，让真正重要的信息被挤压。

例如：

```text
cat 一个 20,000 行日志
读取整个 dist 目录生成文件
把完整 package-lock 当上下文
```

这些内容可能让 Claude 看见很多东西，但不一定让它看见对的东西。

### 4.2 注意力污染

即使 context 空间够，大量无关输出也会增加判断噪音。

【作者推导】这里的“噪音”不是纯比喻。Transformer 模型生成每个 token 时，会对 context 中的内容做 attention。相关信息和无关信息一起进入窗口后，模型需要在更多 token 之间分配注意力。内容越多、越杂，关键线索越容易被稀释，尤其是当无关内容看起来也很像“错误”“配置”“日志”时。

这就是为什么“给更多上下文”不等于“更容易答对”。好的工具使用方式不是把所有东西都塞进来，而是逐步缩小搜索空间，让真正相关的证据更突出。

例如，一个错误日志里有 200 条 warning，真正的 failure 只有一行。如果不先过滤，Claude 可能把时间花在无关 warning 上。

更好的做法是：

```text
不要：把整份日志贴给 Claude
可以：让 Claude 搜索 ERROR / FAIL / stack trace
可以：先运行 grep 过滤关键行
可以：让 Claude 读取日志片段而不是全量内容
```

【作者推导】Claude Code 的工具系统不是鼓励你“给越多越好”，而是鼓励你“用工具逐步缩小不确定性”。

Bash 输出超过限制后保存到文件，也是这种权衡的体现：完整输出不能直接丢，否则 Claude 可能失去必要证据；但完整塞进 context 又会严重压缩后续推理空间。所以更合理的方式是先给路径和预览，后续需要时再让 Claude 有针对性地读取、搜索或分段查看。

---

## 五、工具权限是行动边界，不是弹窗麻烦

工具越强，权限越重要。

【官方事实】Tools reference 明确标注了哪些工具需要权限。例如 Bash、Edit、Write、WebFetch、WebSearch 等通常需要权限，而 Read、Glob、Grep、LSP 等读取或搜索类工具通常不需要权限。工具还可以通过 permissions.allow、permissions.deny、CLI flags、Agent SDK 配置、subagent frontmatter、skill frontmatter、hook matcher 等方式控制。

先把官方工具权限要求落到一张表里：

| 权限要求 | 典型工具 | 含义 |
| --- | --- | --- |
| 通常无需权限 | Read、Glob、Grep、LSP diagnostics / definitions / references | 主要读取项目或语言服务器信息，风险集中在 context 暴露和误读 |
| 通常需要权限 | Edit、Write、NotebookEdit、Bash、PowerShell、WebFetch、WebSearch | 可能改文件、运行命令、访问外部域名或产生副作用 |
| 取决于配置和上下文 | Agent / Task、MCP tools、skills、hooks 触发的工具 | 具体风险取决于工具内部能力、server 权限、skill 内容和 hook 配置 |

这张表不是权限配置模板，而是判断起点。真正配置时还要看项目环境、组织规则、敏感文件、网络边界和 CI / 生产权限。

【作者推导】可以先把工具按风险分成三类：

| 风险层级 | 工具类型 | 风险 |
| --- | --- | --- |
| 低风险 | Read、Glob、Grep、LSP | 主要是泄露或污染 context |
| 中风险 | WebFetch、WebSearch、Skill、Agent | 可能访问外部域名或加载额外流程 |
| 高风险 | Edit、Write、Bash、PowerShell、NotebookEdit | 可能修改文件、运行命令、产生副作用 |

但只按“高 / 中 / 低”还不够。配置 permissions 时，更重要的是看风险性质：

| 工具 | 主要风险 | 判断原则 |
| --- | --- | --- |
| Edit | 局部修改文件 | 关注 diff 范围、是否已读取当前文件、是否有测试验证 |
| Write | 整体创建或覆盖文件 | 覆盖已有文件时风险高于 Edit，应更谨慎 |
| NotebookEdit | 修改 notebook cell | 关注 cell 内容、执行顺序、隐藏状态和运行结果 |
| Bash / PowerShell | 副作用不可完全预测 | 关注是否会改文件、访问网络、删除数据、触发部署或长时间运行 |
| WebFetch / WebSearch | 外部访问和信息外流 | 关注域名、请求内容、是否包含敏感信息 |
| MCP / Agent | 外部能力和间接行动 | 关注 server 权限、工具范围、是否会跨系统产生副作用 |

因此，permissions 的优先级不是简单地“高风险全部拒绝”。更实际的策略是：允许低风险读取，审慎确认写入，严格控制命令副作用和外部系统访问。

权限不是为了打断你，而是为了让 agentic loop 有边界。

如果没有权限边界，一个能搜索、能编辑、能运行命令的 agent 就很难治理。

正确的姿势不是“所有都禁止”，也不是“所有都自动允许”，而是按场景分层：

- 日常读取和搜索：可以相对宽松
- 写文件：看 diff 和范围
- Bash：关注命令是否会修改外部状态
- Web：关注域名和数据外发
- MCP：关注外部系统权限
- CI / 生产环境：用最小权限和审计日志

权限系统会在第 4 篇展开。这里先记住一句话：

> 工具让 Claude Code 能行动，权限决定行动边界。

---

## 六、常见坑和修正

### 坑 1：把工具清单当能力边界

错误认知：

```text
有 Bash，所以 Claude Code 什么命令都能顺利跑。
```

修正：

```text
Bash 能否运行，取决于本地环境、权限、超时、工作目录、依赖是否安装、网络是否可用。
```

工具提供的是行动通道，不保证行动一定成功。

### 坑 2：搜索太宽，读取太多

错误用法：

```text
把整个项目都读一遍，再告诉我问题在哪。
```

修正：

```text
先搜索错误关键词和相关入口文件。
只读取最相关的实现、测试和配置。
如果需要扩大范围，先说明理由。
```

### 坑 3：用 Bash 代替所有工具

有些用户喜欢让 Claude Code 直接 `cat`、`grep`、`sed` 一把梭。

这不一定错，但会绕开部分工具行为和结构化结果。比如 Edit 的 read-before-edit 要求、Grep 的输出模式、Read 的分段读取，都比随手 Bash 更可控。

修正：

- 找文件：优先 Glob
- 找内容：优先 Grep
- 读关键文件：优先 Read
- 改文件：优先 Edit
- 验证：再用 Bash / PowerShell

### 坑 4：把 WebFetch 当原文引用

WebFetch 会抓取并处理网页，Claude 收到的是抽取结果，不一定是完整原文。

修正：

- 关键事实记录 URL 和核查日期
- 对版本、参数、权限等细节回到官方文档
- 不把 WebFetch 摘要当成不可错的原文

### 坑 5：忽略输出规模

错误用法：

```text
分析这个完整日志。
```

如果日志很大，更好的写法是：

```text
请先搜索 ERROR、FAIL、Exception 和 stack trace。
只读取最相关的上下文片段。
如果需要全量日志，先说明原因。
```

修正点：先过滤，再读取；先定位，再展开。

### 坑 6：不区分文本搜索和代码智能

Grep 能找到字符串，不等于理解符号关系。

例如同名函数、重载、接口实现、跨文件引用，纯文本搜索容易误判。

修正：

- 有 LSP 时，让 Claude 用跳转定义、查引用、类型检查
- 没有 LSP 时，用 Grep + Read + 测试补足
- 不把一次搜索结果直接当成全局事实

---

## 七、实践练习

【实践练习】下面的练习不是为了证明 Claude Code 一定会按某种固定路径行动，而是帮助你观察工具调用如何动态改变 context 和下一步判断。

每个练习至少保留一种可追溯产物：命令行输出、会话截图、`/context` 对比、git diff、配置文件 diff，或一个最小复现仓库。

### 练习 1：比较“解释”和“修复”的工具链路

**输入**：对同一个小模块分别提问“解释一下这个模块做什么”和“修复这个模块里的一个边界条件 bug，并验证”。  
**预期观察**：解释任务主要使用读取和搜索工具；修复任务更可能进入 Edit、Bash、LSP 等编辑和验证链路。  
**验证标准**：保存两次任务的工具调用摘要或会话截图，记录工具类型、顺序和数量差异。

### 练习 2：观察工具输出如何进入 context

**输入**：让 Claude Code 先运行一个会失败的测试，再让它根据错误输出定位问题。  
**预期观察**：Claude 的下一步判断会引用测试输出中的错误信息、文件名、断言或堆栈。  
**验证标准**：保存测试命令输出和后续工具调用摘要，说明哪一段输出改变了下一步决策。

### 练习 3：观察大日志对 context 的影响

**输入**：准备一份较长日志，分别让 Claude Code “直接分析全量日志”和“先搜索 ERROR / FAIL / stack trace 再分析”。  
**预期观察**：先过滤的方式更容易聚焦关键错误，也更少引入无关上下文。  
**验证标准**：保存 `/context` 前后对比、关键命令输出或截图，说明两种方式的 context 占用和结论质量差异。

### 练习 4：比较 Grep 和 LSP

**输入**：在一个支持 code intelligence 的项目里，让 Claude 分别用文本搜索和代码智能查找某个函数的定义与引用。  
**预期观察**：Grep 会返回文本匹配；LSP 更能体现符号定义、引用和类型关系。  
**验证标准**：保存两组结果，说明哪种方式更适合当前问题。如果项目没有 LSP，就改做替代观察：用 Grep 找函数名或接口名，用 Read 读取定义和调用点，再运行相关测试或类型检查确认影响范围；保存 Grep 结果、关键 Read 片段和测试输出。

---

## 实践复盘建议

做完练习后，建议按三类整理：

- 与官方文档一致：工具使用是否形成了 gather context、take action、verify results 的循环？工具权限是否和 Tools reference 的描述一致？
- 与预期不同：是权限弹窗比预期频繁、Bash 超时、输出被截断、LSP 没有生效、Grep 搜到太多噪音，还是提示太模糊？
- 文档没有覆盖：保留截图、命令输出、`/context` 对比或最小复现仓库，后续 Claude Code 版本升级后再复测。

这篇文章的核心结论可以压成一句话：

```text
Claude Code 不是因为“会说”而能做事，
而是因为工具结果不断进入 context，
让它能在真实反馈中持续修正下一步行动。
```

---

## 参考资料

### 官方主参考

- [Tools reference](https://code.claude.com/docs/en/tools-reference)
- [How Claude Code works](https://code.claude.com/docs/en/how-claude-code-works)
- [Common workflows](https://code.claude.com/docs/en/common-workflows)
- [Explore the context window](https://code.claude.com/docs/en/context-window)

### GitHub / 社区补充

- [anthropics/claude-code](https://github.com/anthropics/claude-code)
- [subinium/awesome-claude-code](https://github.com/subinium/awesome-claude-code)

社区仓库更适合用来观察别人如何组织 workflows、skills、hooks、MCP 示例和实践清单，不用于替代官方工具行为定义。

社区资料只用于补充实践案例，不用于覆盖官方行为定义。凡是社区经验与官方文档冲突，以官方文档和本地实践观察为准。
