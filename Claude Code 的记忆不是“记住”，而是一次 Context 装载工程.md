# Claude Code 的记忆不是“记住”，而是一次 Context 装载工程

> 文档核查日期：2026-05-18  
> 作者：数据室 伍涛。  
> 写作初衷：在斌总和葛处的鼓励与倡导下，围绕 Claude Code 做一次面向底层原理的系统学习，把学习过程中的理解、积累和实践经验沉淀下来，并以文章形式分享给更多同事和读者。  
> 适合读者：个人开发者、团队负责人  
> 本文定位：Claude Code 系列第 2 篇，聚焦 instructions / memory / context 的装载机制。  
> 实践建议：本文末尾提供动手练习，建议读者结合自己的 Claude Code 版本和配置复现观察。

---

## 先把边界说清楚

这篇文章只讲一个问题：

> Claude Code 为什么能在一次新的 session 里重新获得项目上下文？

它不讲 Claude Code 的完整工具系统，也不展开 Bash、Edit、Grep、MCP tool 调用结果如何持续追加进 context。那些属于“工具系统如何驱动 agent loop”的话题，应该放到下一篇。

本文只关注 context 的“装载来源”：

- CLAUDE.md
- CLAUDE.local.md
- `.claude/rules/`
- Auto memory
- Skills 的描述与被调用后的正文
- MCP tool names 与 tool definitions 的加载时机
- system prompt、output style、`--append-system-prompt`
- `/compact` 后哪些内容会重注入

【官方事实】严格按官方说法，Claude Code 的跨会话记忆主要由两套机制承担：

- CLAUDE.md files：你写的持久指令
- Auto memory：Claude 根据你的纠正和偏好自动写下的笔记

但从一次 session 的 context 装载工程看，还必须同时理解 rules、skills、MCP、system prompt、hooks 这些周边机制。它们不都叫“记忆”，但都会影响 Claude 在当前会话中“看见什么、遵循什么、能调用什么”。

本文只在容易混淆的地方标注来源类型：

- 【官方事实】：来自 Claude Code 官方文档或官方仓库。
- 【作者推导】：基于官方事实整理出的工程判断。
- 【实践练习】：读者可以在自己的环境中复现观察的部分。

---

## 一、核心认知：每次 session 都是新的 context window

每次启动 Claude Code，都是一个新的 context window。上一轮对话里你说过的话、Claude 读过的文件、工具输出、临时约束，默认不会自动出现在下一次会话中。

Claude Code 解决这个问题的方式，不是让模型权重发生变化，也不是让模型“永久记住你”，而是每次启动时重新装载一组上下文。

可以把它理解成：

```text
记忆效果 = 启动时装载 + 工作中按需读取 + compact 后重注入
```

【官方事实】官方 context window 文档把一次 session 的 context 填充过程拆成几个阶段：

- 启动前：CLAUDE.md、auto memory、MCP tool names、skill descriptions 等进入 context
- 工作中：文件读取、工具输出、路径规则、hook 触发等继续改变上下文
- subagent：大范围研究可以在独立 context window 中完成，只把 summary 回传主会话
- compact 后：部分启动内容重新从磁盘注入，部分按需规则需要再次触发

【作者推导】这里最重要的不是某个固定 token 数字，而是这个事实：

> context 是有限且竞争的资源。指令、记忆、文件内容、工具输出和模型回复都在争同一个窗口。

具体可用空间会随模型、Claude Code 版本、配置、MCP、skills、output style 等变化。官方 context window 页面中的 token 数是 representative examples，真实用量应以 `/context` 为准。

---

## 二、启动时到底装了什么

在你输入第一句话之前，Claude Code 已经把一部分内容放进了 context。

| 内容 | 加载行为 | 说明 |
| --- | --- | --- |
| System prompt / output style | 启动时进入系统提示层 | 包括 Claude Code 内置行为约束、output style、`--append-system-prompt` 等 |
| CLAUDE.md / CLAUDE.local.md | 启动时全量加载 | 当前工作目录向上查找并拼接；子目录 CLAUDE.md 按需加载 |
| Unscoped rules | 启动时加载 | `.claude/rules/` 中没有 `paths` frontmatter 的规则等同于全局项目指令 |
| Path-scoped rules | 读取匹配文件时加载 | 有 `paths:` frontmatter 的规则只在 Claude 处理匹配文件时进入 context |
| Auto memory | 启动时加载索引 | `MEMORY.md` 前 200 行或前 25KB，取先达到的限制 |
| Skills 描述 | 启动时可见 | skill body 不会启动时全量加载，只有描述帮助 Claude 判断何时使用 |
| MCP tool names | 启动时加载 | 官方当前文档说明默认只加载 tool names |
| MCP tool definitions | 默认延迟加载 | Tool Search 默认启用，definitions 在 Claude 需要时按需进入 context |

### MCP 这里要更新旧认知

旧版本或旧经验里，常见说法是：接入很多 MCP server 会把所有工具名和描述都在启动时塞进 context，因此 MCP 是最大的隐性开销。

按 2026-05-17 核查的官方 MCP 文档，这个说法需要修正。

官方当前说明是：

- Tool Search 默认启用
- MCP tools 默认 deferred，不在启动时把完整 definitions 全量装进 context
- 只有 tool names 在 session start 加载
- Claude 需要某类工具时，会通过 tool search 发现相关工具
- 只有实际使用的 MCP tools 会进入 context

但有几个例外：

- `ENABLE_TOOL_SEARCH=false` 会禁用延迟加载，改成 upfront loading
- `ENABLE_TOOL_SEARCH=auto` 或 `auto:N` 会按 context 百分比阈值决定是否前置加载
- 某个 server 配置 `alwaysLoad: true` 后，该 server 的工具会在启动时进入 context
- Vertex AI、非一方 `ANTHROPIC_BASE_URL` 代理、模型不支持 `tool_reference` blocks 等情况下，行为可能 fallback

因此本文不再写“MCP 工具描述一定启动时全量加载”。更准确的说法是：

> MCP tool names 通常会启动时加载；tool definitions 在官方当前默认配置下会延迟加载。实际 context 成本要用 `/context` 和 `/mcp` 在本地验证。

---

## 三、/compact 后哪些内容会留下

`/compact` 会把长对话历史压缩成结构化摘要，给后续工作腾出 context 空间。但不同来源的内容，compact 后的命运不一样。

| 机制 | `/compact` 后行为 |
| --- | --- |
| System prompt / output style | 不变；它们不属于普通 message history |
| 项目根 CLAUDE.md | 从磁盘重新读取并重注入 |
| Unscoped rules | 从磁盘重新读取并重注入 |
| Auto memory | 从磁盘重新读取并重注入 |
| Path-scoped rules | 丢失，直到再次读取匹配文件 |
| 子目录 CLAUDE.md | 丢失，直到再次读取该子目录文件 |
| 已调用 skill body | 重注入，但每个 skill 最多保留前 5,000 tokens，总预算 25,000 tokens |
| Hooks | 不适用；hook 是代码执行，不是 context |
| 对话里的临时指令 | 被摘要吸收或丢失，不应当依赖它长期生效 |

这解释了一个常见问题：

> 你在对话里说“这次不要动 X 文件”，compact 之后它可能不再可靠。

如果一条规则需要跨 compact 保持稳定，应该写进项目根 CLAUDE.md、unscoped rule，或者用 hook / permission 做确定性控制。

如果某条规则只在某个目录下有意义，用 path-scoped rule 是对的，但要接受它在 compact 后需要通过再次读取匹配文件来恢复。

---

## 四、记忆系统的边界：它是软约束，不是强制配置

官方文档明确说明：CLAUDE.md 和 Auto memory 会被 Claude 当作 context，而不是不可违反的配置。

这意味着：

- 它们能提高 Claude 遵循项目约定的概率
- 但不能保证每次 100% 执行
- 写得越具体、越短、越结构化，越容易被遵循
- 模糊、冲突、过长的规则会降低可靠性

如果某件事必须稳定发生，不应该只写在 CLAUDE.md 里。

| 需求 | 更合适的机制 |
| --- | --- |
| 每次提交前必须跑测试 | CI / Git hooks / Claude Code hooks |
| 禁止读取 `.env` | permissions.deny / sensitive files rules |
| 禁止危险 Bash 命令 | permissions / managed settings / PreToolUse hook |
| 统一格式化代码 | formatter / lint / PostToolUse hook |
| 组织级安全规则 | managed settings / managed CLAUDE.md |
| 临时任务背景 | 单独文件显式读取，不要写进长期 CLAUDE.md |

一句话：

> CLAUDE.md 和 Auto memory 负责“引导默认行为”，hooks、permissions、CI 负责“确定性约束”。

---

## 五、四类上下文机制怎么选

这里不要把所有东西都叫“记忆”。更准确的分类是：

| 机制 | 谁写 | 加载时机 | 适合放什么 | 风险 |
| --- | --- | --- | --- | --- |
| CLAUDE.md | 你或团队 | 启动时全量加载 | 每次会话都需要的项目事实、命令、约定 | 写太长会占 context，且遵循率下降 |
| Auto memory | Claude 自动写 | 启动时加载 `MEMORY.md` 前 200 行或 25KB | Claude 在使用中学到的偏好、调试经验、项目规律 | 可能写错，需要审计 |
| `.claude/rules/` | 你或团队 | 无 `paths` 时启动加载；有 `paths` 时按需加载 | 模块或文件类型专属规则 | 路径匹配不准会导致没触发 |
| Skills | 你、团队或插件 | 描述可见；body 仅在使用时加载 | 长流程、参考资料、重复 checklist、可调用工作流 | 描述写不好会不触发或误触发 |

【作者推导】判断逻辑可以这样用：

- 删掉这条，Claude 每次都会犯同一个错：放 CLAUDE.md
- 它只在某个目录或文件类型下有意义：放 `.claude/rules/` 并加 `paths`
- 它是一个多步骤流程，不是每次都用：做成 Skill
- 它是 Claude 在互动中自然学到的偏好或调试经验：交给 Auto memory，但定期审计
- 它必须被强制执行：不要只写进 memory，用 hooks、permissions、CI 或 managed settings

---

## 六、CLAUDE.md：你写给 Claude 的项目说明书

CLAUDE.md 是最基础的持久指令文件。它不是系统提示的一部分，而是作为用户上下文进入 session。

### 6.1 文件放在哪里

官方文档列出的主要位置如下：

| 层级 | 位置 | 作用 |
| --- | --- | --- |
| Managed policy | macOS: `/Library/Application Support/ClaudeCode/CLAUDE.md`；Linux/WSL: `/etc/claude-code/CLAUDE.md`；Windows: `C:\Program Files\ClaudeCode\CLAUDE.md` | 组织级统一指令 |
| User instructions | `~/.claude/CLAUDE.md` | 个人全局偏好 |
| Project instructions | `./CLAUDE.md` 或 `./.claude/CLAUDE.md` | 当前项目团队共享 |
| Local instructions | `./CLAUDE.local.md` | 当前项目个人偏好，建议加入 `.gitignore` |
| Subdirectory instructions | 子目录下的 `CLAUDE.md` / `CLAUDE.local.md` | 读取该子目录文件时按需加载 |

加载规则：

- Claude Code 会从当前工作目录向上查找 CLAUDE.md 和 CLAUDE.local.md
- 找到的文件会拼接进 context，不是互相覆盖
- 加载顺序从更广的目录到更具体的目录
- 同一目录下，CLAUDE.local.md 会追加在 CLAUDE.md 后面
- 子目录里的 CLAUDE.md 不在启动时加载，而是在 Claude 读取该子目录文件时加载

### 6.2 怎么写才更有效

官方建议很明确：短、具体、结构化、一致。

实践上可以用这几个原则：

- 控制大小：目标 200 行以内，越短越容易遵循
- 写具体：`Run npm test before committing` 比 `记得测试` 有效
- 用结构：标题和列表比大段文字更容易扫描
- 避免冲突：不同层级的 CLAUDE.md、rules、skills 之间不要互相打架
- 不写临时状态：Sprint 临时禁区、一次性任务背景，不要塞进长期 CLAUDE.md

一个项目根 CLAUDE.md 可以这样组织：

```markdown
# CLAUDE.md

## Project Overview
- This is a TypeScript Next.js application.

## Commands
- Typecheck: `pnpm typecheck`
- Unit test single file: `pnpm test -- --testPathPattern=<file>`
- Build: `pnpm build`

## Architecture
- API routes live in `app/api/`.
- Database access goes through `lib/db/`.
- Environment variables must be read through `lib/env.ts`.

## Completion Criteria
- Typecheck passes.
- Relevant tests pass.
- No unrelated files changed.
```

### 6.3 `@` import 不会节省 context

CLAUDE.md 支持 `@path/to/file` 引入其他文件。这个机制适合组织内容，但不要把它误解成“按需加载”。

官方说明是：被 `@` 引入的文件会在启动时展开并进入 context。也就是说：

```markdown
@docs/deploy.md
```

并不会节省 context，它只是把另一个文件的内容复制进启动上下文。

如果一份文档不是每次都需要，更好的写法是：

```markdown
当处理部署问题时，先读取 docs/deploy.md。
```

这样 Claude 只有在任务相关时才会通过文件工具读取它。

`@` 更适合的场景是：多个 worktree 共享同一份个人配置。例如每个 worktree 的 `CLAUDE.local.md` 只写：

```markdown
@~/.claude/my-project-instructions.md
```

### 6.4 AGENTS.md 不是 Claude Code 的默认入口

官方文档说明：Claude Code 默认读 `CLAUDE.md`，不是 `AGENTS.md`。

如果你的仓库已经有 AGENTS.md，可以在 CLAUDE.md 中导入它：

```markdown
@AGENTS.md

## Claude Code
- Use plan mode for changes under `src/billing/`.
```

这样可以避免同时维护两份 agent instructions。

### 6.5 HTML 注释不会进入注入后的上下文

CLAUDE.md 中的块级 HTML 注释会在注入 context 前被剥除，适合给人类维护者留说明：

```markdown
<!-- Maintainer note: update this section when test commands change. -->
- Test command: `pnpm test`
```

但代码块里的注释会保留，Claude 仍然能看到。

---

## 七、`.claude/rules/`：路径作用域规则

大型项目里，不要把所有规则都塞进项目根 CLAUDE.md。官方推荐用 `.claude/rules/` 做模块化管理。

没有 `paths` frontmatter 的 rule，会在启动时无条件加载：

```markdown
# Testing Rules

- Use `pnpm test` for unit tests.
- Prefer targeted tests over full test suite.
```

带 `paths` frontmatter 的 rule，只在 Claude 处理匹配文件时加载：

```markdown
---
paths:
  - "src/payments/**"
---

# Payment Rules

- All money amounts use integer cents.
- Do not use floating point math in payment flows.
- After changing payment logic, run `pnpm test:payments`.
```

关键点：

- 字段名是 `paths`
- 支持 glob patterns
- 可以写多个 pattern
- `.md` 文件会递归发现
- rules 支持 symlink
- user-level rules 可以放在 `~/.claude/rules/`
- path-scoped rules 在 compact 后不会自动重注入，需要再次读取匹配文件

如果一条规则必须跨 compact 稳定存在，不要用 path-scoped rule。把它放到项目根 CLAUDE.md 或 unscoped rule，或者用 hook / permission 做强约束。

---

## 八、Auto memory：Claude 自己写的笔记

【官方事实】Auto memory 是 Claude Code 的自动笔记系统。它会根据你在会话中的纠正、偏好、项目规律、调试经验，判断哪些内容未来有用，然后写入本地 memory 文件。

### 8.1 版本和开关

【官方事实】官方文档说明：Auto memory 需要 Claude Code v2.1.59 或更高版本。

截至 2026-05-18，官方 memory 文档没有把 Auto memory 标注为 experimental、preview 或 beta。本文按正式功能描述，但仍建议用 `/memory` 和本地配置确认你当前版本中的实际行为。

查看版本：

```bash
claude --version
```

Auto memory 默认开启。可以通过 `/memory` 切换，也可以在项目设置里关闭：

```json
{
  "autoMemoryEnabled": false
}
```

也可以通过环境变量临时禁用：

```bash
CLAUDE_CODE_DISABLE_AUTO_MEMORY=1 claude
```

### 8.2 存储位置

【官方事实】每个项目有自己的 memory 目录：

```plain
~/.claude/projects/<project>/memory/
├── MEMORY.md
├── debugging.md
├── api-conventions.md
└── ...
```

重要细节：

- `<project>` 从 git repository 派生
- 同一 repo 的 worktrees 和子目录共享同一份 auto memory
- 不在 git repo 内时，使用项目根目录派生
- memory 是本机本地文件，不跨机器或云环境同步
- `MEMORY.md` 是索引，启动时只加载前 200 行或 25KB
- 主题文件不会启动时加载，Claude 需要时才用文件工具读取

如需自定义存储位置，可以在用户级设置里配置：

```json
{
  "autoMemoryDirectory": "~/my-custom-memory-dir"
}
```

这个配置不能放在项目或 local settings 中，防止仓库把 memory 重定向到敏感位置。

### 8.3 Auto memory 会记什么

【官方事实】官方给出的典型内容包括：

- build commands
- debugging insights
- architecture notes
- code style preferences
- workflow habits

它不是每次会话都写，而是由 Claude 判断这条信息未来是否有用。

你也可以明确告诉 Claude：

```plain
记住：这个项目始终用 pnpm，不用 npm。
```

也可以让它忘掉：

```plain
忘掉之前关于 npm 的项目规则。
```

但 Auto memory 是 Claude 写的，不一定完全准确。所以需要定期审计。

【作者推导】建议：

- 新项目前几次会话后打开 `/memory` 检查
- 发现错误立即改掉或让 Claude 忘掉
- 不要把 Auto memory 当成团队共享知识库
- 团队规则应写进项目 CLAUDE.md 或 rules，并走 PR

---

## 九、Skills：不是记忆系统，但和 context 管理强相关

Skills 在官方文档中属于 Tools and plugins 分区，不是 memory 页面里的子系统。

但它和本文有关，因为它解决的是同一个问题：

> 不要把偶尔才用到的长流程塞进启动 context。

官方说明：skill body 只有在被使用时才加载。启动时主要依靠 skill description 帮助 Claude 判断何时使用。

### 9.1 什么时候该做成 Skill

适合做 skill 的内容：

- 你经常复制进聊天框的 checklist
- 多步骤部署流程
- 复杂 code review 流程
- 长参考资料
- 需要动态注入上下文的工作流
- 需要复用的命令式任务

不适合做 skill 的内容：

- 每次会话都必须知道的项目事实
- 必须强制执行的安全规则
- 临时状态
- 只有一句话的偏好

### 9.2 一个最小 Skill 示例

```markdown
---
description: Summarizes uncommitted changes and flags risks. Use when the user asks what changed or requests a diff review.
---

## Current changes

!`git diff HEAD`

## Instructions

Summarize the changes above in two or three bullet points, then list risks such as missing error handling, hardcoded values, or tests that need updating.
```

其中：

- 目录名会成为 `/skill-name`
- `description` 帮 Claude 判断何时自动使用
- `!`command`` 会在 Claude 看到 skill 内容前执行，把输出注入 prompt

### 9.3 控制谁能触发 Skill

如果不希望 Claude 自动触发某个 skill，可以用：

```yaml
disable-model-invocation: true
```

这样只有用户显式输入 `/skill-name` 时才会运行。

### 9.4 compact 后 Skill 如何保留

官方 Skills 文档说明：

- compact 后会重新附加最近调用过的 skill
- 每个 skill 最多保留前 5,000 tokens
- 所有 re-attached skills 共用 25,000 tokens 预算
- 从最近调用的 skill 开始填充，较早的可能被丢弃

所以长 skill 应该把最重要的说明放在文件开头。

---

## 十、常见坑和修正

### 坑 1：把 CLAUDE.md 当配置文件

错误认知：

```markdown
只要写进 CLAUDE.md，Claude 就一定会执行。
```

正确认知：

```markdown
CLAUDE.md 是 context，不是强制配置。
```

必须执行的动作，用 hook、permission、CI、formatter、lint。

### 坑 2：什么都往 CLAUDE.md 里塞

CLAUDE.md 超过 200 行后，会消耗更多 context，也可能降低遵循率。

修正：

- 常驻事实：CLAUDE.md
- 模块规则：`.claude/rules/`
- 长流程：Skills
- 强制行为：Hooks / permissions / CI
- 临时背景：单独文件显式读取

### 坑 3：用 `@` import 以为能省 token

`@docs/long-guide.md` 会在启动时展开进入 context。它适合组织文件，不适合省 context。

如果只是偶尔用到，写触发条件，让 Claude 需要时再读。

### 坑 4：把临时状态写进长期记忆

例如：

```markdown
本周暂时不要改 billing 模块。
```

这类信息不该进入长期 CLAUDE.md。可以放进 `sprint-context.md`，会话开始时显式读取，用完删除。

### 坑 5：Auto memory 写错后不审计

Auto memory 是 Claude 自动写的，本质上是可编辑的 markdown，不是权威事实库。

修正：

- 新项目早期频繁检查 `/memory`
- 错误记忆立即删除或修正
- 团队规则不要只依赖个人 auto memory

### 坑 6：沿用旧 MCP context 结论

旧结论：

```markdown
MCP 工具名和描述都会启动时全量加载，所以 MCP 是最大的启动 context 开销。
```

当前官方文档下更准确的写法：

```markdown
MCP tool names 启动时加载；tool definitions 默认通过 Tool Search 延迟加载。实际成本取决于 ENABLE_TOOL_SEARCH、alwaysLoad、模型支持、代理环境等配置，建议用 /context 和 /mcp 在自己的环境中观察。
```

---

## 十一、实践练习

【实践练习】如果你想把本文的结论变成自己的经验，可以做下面 5 个练习。每个练习都建议留下命令输出、截图或简短记录，方便半年后对照 Claude Code 版本变化。

### 练习 1：观察启动 context 基线

**输入**：在一个干净项目中启动 Claude Code，立即运行 `/context` 和 `/memory`。  
**预期观察**：能看到启动前装载的上下文来源，例如 CLAUDE.md、Auto memory、skill descriptions、MCP tool names 等。  
**验证标准**：保存一张 `/context` 截图或一段关键输出摘录，并能说明这些内容分别属于哪类启动上下文。

### 练习 2：比较 CLAUDE.md 和 `@` import 成本

**输入**：分别使用短 CLAUDE.md、长 CLAUDE.md、包含 `@docs/long-guide.md` 的 CLAUDE.md，启动后运行 `/context`。  
**预期观察**：长文件和被 `@` 引入的文件都会增加启动 context；`@path` 更像内容展开，而不是按需加载。  
**验证标准**：整理一张三列表格，包含配置、context 变化和你的结论。

### 练习 3：验证 path-scoped rules 触发

**输入**：创建 `.claude/rules/payments.md`，设置 `paths: ["src/payments/**"]`，然后让 Claude 读取匹配和不匹配文件。  
**预期观察**：无 `paths` 的 rule 会作为全局项目指令加载；带 `paths` 的 rule 只应在处理匹配路径时加载。  
**验证标准**：记录“触发”和“未触发”两组结果，并说明你观察到的加载时机。

### 练习 4：观察 compact 后重注入

**输入**：在长会话中读取 path-scoped rule、调用 skill、运行 `/compact`。  
**预期观察**：项目根 CLAUDE.md 和 Auto memory 会重新注入；path-scoped rule 通常需要再次读取匹配文件；skill body 受重附加预算限制。  
**验证标准**：整理 compact 前后 `/context` 对比，并标出哪些内容自动回来、哪些内容需要再次触发。

### 练习 5：观察 MCP Tool Search

**输入**：启用多个 MCP server，分别在默认配置、`ENABLE_TOOL_SEARCH=false`、`alwaysLoad: true` 下运行 `/context` 和 `/mcp`。  
**预期观察**：默认情况下 tool definitions 可能延迟加载；禁用 Tool Search 或设置 alwaysLoad 后，启动成本可能上升。  
**验证标准**：记录每种配置下 MCP 相关 context 成本，并写一句自己的判断。

---

## 实践复盘建议

做完练习后，不要只记录“对”或“错”。更有价值的是把观察结果分成三类：

- 与官方文档一致：可以作为你当前版本下的可靠经验。
- 与预期不同：优先检查 Claude Code 版本、配置、MCP server、环境变量和权限模式。
- 文档没有覆盖：保留截图或命令输出，后续版本升级后再复测。

---

## 参考资料

### 官方主参考

- [How Claude remembers your project](https://code.claude.com/docs/en/memory)
- [Explore the context window](https://code.claude.com/docs/en/context-window)
- [Explore the .claude directory](https://code.claude.com/docs/en/claude-directory)
- [Extend Claude with skills](https://code.claude.com/docs/en/skills)
- [Connect Claude Code to tools via MCP](https://code.claude.com/docs/en/mcp)
- [Automate workflows with hooks](https://code.claude.com/docs/en/hooks-guide)
- [Debug your configuration](https://code.claude.com/docs/en/debug-your-config)

### GitHub / 社区补充

- [anthropics/skills](https://github.com/anthropics/skills)
- [subinium/awesome-claude-code](https://github.com/subinium/awesome-claude-code)

社区资料只用于补充实践案例，不用于覆盖官方行为定义。凡是社区经验与官方文档冲突，以官方文档和本地实测为准。

