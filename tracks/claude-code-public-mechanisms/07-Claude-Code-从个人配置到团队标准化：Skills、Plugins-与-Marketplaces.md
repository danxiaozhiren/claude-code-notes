# 从个人配置到团队标准化：Skills、Plugins 与 Marketplaces

> 文档核查日期：2026-05-20  
> 作者：wt  
> 写作初衷：在领导的鼓励与倡导下，围绕 Claude Code 做一次面向底层原理的系统学习，把学习过程中的理解、积累和实践经验沉淀下来，并以文章形式分享给更多同事和读者。  
> 适合读者：个人开发者、团队负责人、平台开发者  
> 本文定位：Claude Code 系列第 7 篇，承接第 5 篇的 Hooks / MCP 和第 6 篇的多 Agent，解释个人经验如何沉淀成可复用、可分发、可治理的团队配置。  
> 实践建议：本文末尾提供动手练习，建议读者从一个最小 skill 开始，再升级为 plugin，最后用本地 marketplace 做一次分发验证。

---

## 先把边界说清楚

前几篇已经讲过 Claude Code 的核心运行机制：context 装载、工具调用、权限边界、Hooks / MCP、多 Agent。现在问题变成：

> 我学到的经验，如何从脑子里的习惯，变成别人也能安装、复用、升级和治理的能力？

这篇讲三个层级：

```text
Skills：把一段知识或工作流做成按需加载的能力。
Plugins：把 skills、agents、hooks、MCP、settings 等打包成可分发单元。
Marketplaces：把多个 plugins 做成团队或社区可发现、可安装、可更新的目录。
```

不要把它们混成一个概念：

```text
Skill 解决“如何复用一段知识或流程”。
Plugin 解决“如何打包一组能力”。
Marketplace 解决“如何分发和治理一组插件”。
```

本文关注：

- Skills 的 reference / task 两种用法
- skill frontmatter、`disable-model-invocation`、`allowed-tools`
- Skills 和 CLAUDE.md 的边界
- Plugin 目录结构、`.claude-plugin/plugin.json`、namespace
- Plugin 能包含 skills、agents、hooks、MCP、LSP、bin、settings、monitors 等组件
- Marketplace 的 `marketplace.json`、plugin source、版本策略和组织治理
- 什么时候从 skill 升级为 plugin

不展开的内容：

- Hooks 和 MCP 的底层机制：见第 5 篇
- Subagents、Agent Teams、Worktrees：见第 6 篇
- GitHub Actions、Headless、Agent SDK：放到第 8 篇

本文只在容易混淆的地方标注来源类型：

- 【官方事实】：来自 Claude Code 官方文档或官方仓库。
- 【官方 GitHub】：来自 Anthropic 官方 GitHub 仓库。
- 【作者推导】：基于官方事实整理出的机制解释和工程判断。
- 【实践练习】：读者可以在自己的环境中复现观察的部分。

---

## 一、核心认知：标准化不是把所有东西塞进 CLAUDE.md

很多人一开始会把所有经验都写进 CLAUDE.md：

```text
项目怎么启动
测试怎么跑
接口怎么写
发布前检查什么
代码审查怎么做
出错时如何排查
```

这很好，说明经验开始沉淀了。

但 CLAUDE.md 不是无限仓库。

【官方事实】Skills 文档明确说，Claude 会根据需要加载 skill，skill body 只有在被使用时才加载；相比之下，CLAUDE.md 更像会话开始时的常驻上下文。官方还建议：当你反复粘贴同一套指令、清单或多步流程，或者 CLAUDE.md 的某一节已经变成过程说明时，就可以创建 skill。

【作者推导】这背后是 context 成本的权衡：

```text
CLAUDE.md：常驻，适合事实和长期规则。
Skill：按需，适合流程、检查清单、长参考材料。
Plugin：分发，适合团队共享的一组能力。
Marketplace：治理，适合团队或社区维护多个插件。
```

所以经验沉淀的升级路径通常是：

```text
一段临时 prompt
  -> 写进 CLAUDE.md
  -> 独立成 skill
  -> 打包成 plugin
  -> 进入 marketplace
  -> 用 managed settings 治理来源和版本
```

【作者推导】每升一级，解决的问题都不同：

| 层级 | 解决什么 | 代价 |
| --- | --- | --- |
| Prompt | 临时表达一次任务 | 不可复用 |
| CLAUDE.md | 常驻项目规则 | 长了会占 context、难维护 |
| Skill | 按需知识和流程 | 需要写触发描述和边界 |
| Plugin | 打包和命名空间 | 需要版本、测试、结构 |
| Marketplace | 分发和治理 | 需要源可信、版本策略、组织策略 |

不要太早上 plugin，也不要让 CLAUDE.md 承担一切。

---

## 二、Skills：把经验做成按需加载的能力

【官方事实】Skills 通过一个 `SKILL.md` 文件定义。Claude Code 会把 skill 加入可用能力集合，用户可以用 `/skill-name` 直接调用，Claude 也可以在相关任务中自动调用。Skill 可以包含支持文件，例如 reference、examples、scripts。

一个最小 skill：

```markdown
---
description: Reviews uncommitted changes and flags risky diffs. Use when the user asks what changed or wants a review before commit.
---

Review the current changes.

Focus on:
- correctness
- missing tests
- accidental secrets
- risky shell or deployment changes
```

### 2.1 reference content 和 task content

【官方事实】Skills 文档把 skill 内容分成两类思路：reference content 和 task content。这里的分类描述的是“内容属性”，不是“触发模式”。默认情况下，用户和 Claude 都可以调用 skill；是否允许 Claude 自动调用，要看 frontmatter 里的调用控制字段。

| 类型 | 适合放什么 | 调用控制判断 |
| --- | --- | --- |
| Reference content | 项目规范、API 约定、领域知识、样式指南 | 通常可以让 Claude 自动识别并加载 |
| Task content | 部署、提交、代码生成、审查、排障等多步流程 | 默认也能自动调用；有副作用时更应该手动控制 |

不要把 reference / task 理解成两种不同的 invocation mode。它们只是内容组织方式。真正改变调用方式的是 `disable-model-invocation` 和 `user-invocable`。

【作者推导】可以这样判断：

```text
如果它是在“做任何相关任务时都应该知道的知识”，更像 reference skill。
如果它是“执行一个明确动作的步骤”，更像 task skill。
```

例如：

- `api-conventions`：reference skill，告诉 Claude 接口命名、错误格式、认证约定。
- `deploy`：task skill，要求按步骤测试、构建、发布、验证。
- `review-diff`：介于两者之间，可以自动触发，也可以手动 `/review-diff`。

### 2.2 frontmatter 控制 skill 怎么出现

【官方事实】Skill 使用 YAML frontmatter 配置行为。官方文档列出多个字段，其中 `description` 推荐填写，用于帮助 Claude 判断何时加载 skill。`name` 可选，省略时使用目录名。

常用字段：

| 字段 | 作用 | 工程判断 |
| --- | --- | --- |
| `name` | 显示名称，省略时用目录名 | 稳定命名，避免以后重命名破坏调用 |
| `description` | 说明 skill 做什么、何时用 | 最重要，写清触发场景 |
| `when_to_use` | 追加触发说明 | 用于补充边界和反例 |
| `argument-hint` / `arguments` | 说明参数 | 适合 `/deploy prod` 这类命令 |
| `disable-model-invocation` | 禁止 Claude 自动调用；description 不进入 context | 适合发布、提交、发消息等带副作用流程 |
| `user-invocable` | 是否出现在用户可调用菜单 | 背景知识可设为 false |
| `allowed-tools` | skill 活跃时预批准工具 | 必须谨慎，避免扩大权限 |
| `context` / `agent` | 在 forked subagent context 中运行 | 适合复杂或污染 context 的任务 |
| `hooks` | skill 生命周期内的 hooks | 适合该流程专属检查 |
| `paths` | 只在匹配文件路径时自动激活 | 适合 monorepo 或特定目录规范 |

【官方事实】`description` 和 `when_to_use` 的组合会在 skill 列表中截断到一定长度以降低 context 成本。官方文档还说明：skill 被调用后，渲染后的 `SKILL.md` 内容会进入对话，并在当前 session 后续 turns 中保留；compact 后最近调用的 skills 会按预算重附加。

【作者推导】这说明 skill 不是“零成本魔法”。它比 CLAUDE.md 更按需，但一旦加载，内容仍会进入 context。因此：

- `description` 要短而准
- `SKILL.md` 主体不要变成百科全书
- 长参考材料放到支持文件，按需引用
- 会产生副作用的 skill 用 `disable-model-invocation: true`

### 2.3 `disable-model-invocation` 是安全边界的一部分

【官方事实】`disable-model-invocation: true` 会阻止 Claude 自动调用该 skill，只允许用户手动调用。官方文档建议把它用于用户想控制时机的 workflow，例如 deploy、commit、send Slack message 等。更关键的是：设置后该 skill 的 description 也不会进入普通 session 的 context；只有用户手动调用时，完整 skill 内容才会加载。

这三个状态的区别是：

| frontmatter 状态 | 用户能调用 | Claude 能自动调用 | context 中默认可见 |
| --- | --- | --- | --- |
| 默认 | 是 | 是 | description 可见，完整内容调用后加载 |
| `disable-model-invocation: true` | 是 | 否 | description 不可见，完整内容手动调用后加载 |
| `user-invocable: false` | 否 | 是 | description 可见，完整内容调用后加载 |

【作者推导】这个字段不是“减少打扰”的小选项，而是安全边界：

```text
让 Claude 自动知道 API 规范，可以。
让 Claude 自动决定发布生产，不行。
```

适合自动调用：

- API 设计规范
- 文档风格
- 测试命名规则
- 项目术语表

适合手动调用：

- `/deploy`
- `/commit`
- `/publish`
- `/send-report`
- `/migrate-db`

如果一个 skill 会触发真实副作用，默认应该手动触发，再配合 permissions、hooks 或 CI gate。它和“把 description 写得保守一点”不是同一个层级：保守 description 只是降低 Claude 误触发概率；`disable-model-invocation` 则让 Claude 在普通上下文里根本看不到这个 skill 的 description。

### 2.4 supporting files：让主入口保持薄

【官方事实】Skills 可以包含多个文件。官方建议让 `SKILL.md` 保持核心说明，把大型 reference docs、examples、scripts 放在旁边，并在 `SKILL.md` 中引用这些文件，告诉 Claude 何时加载。

示意结构：

```text
code-review/
├── SKILL.md
├── reference.md
├── examples.md
└── scripts/
    └── collect-diff.sh
```

【作者推导】这解决的是“可维护性”和“context 成本”的双重问题：

- `SKILL.md` 是目录和决策入口
- `reference.md` 是详细知识
- `examples.md` 是少量高质量样例
- `scripts/` 是可执行辅助，不必全量进入 context

好的 skill 像工具箱，不像一整本手册塞进提示词。

---

## 三、Skills 和 CLAUDE.md：不是谁替代谁

第 2 篇已经讲过 CLAUDE.md 是静态 context 装载来源。这里换个角度看：什么时候该从 CLAUDE.md 拆出 skill？

【作者推导】可以用这张表判断：

| 内容 | 放 CLAUDE.md | 放 Skill |
| --- | --- | --- |
| 项目是什么、怎么启动 | 适合 | 不必 |
| 全局代码风格 | 适合 | 如果很长可拆 reference skill |
| 发布流程 | 不适合常驻 | 适合 task skill |
| PR 审查清单 | 可以简写 | 适合 review skill |
| 长 API 文档 | 不适合 | 放 skill 支持文件 |
| 团队故障排查流程 | 简述入口 | 适合 debug skill |
| 一次性临时规则 | 不适合 | 也不适合，直接 prompt |

一个常见演化：

```text
CLAUDE.md:
  "提交前请检查测试和风险。"

Skill:
  /review-diff
  读取 diff、按清单审查、输出风险、给出 commit message 建议。
```

【作者推导】这背后的原则是：

```text
CLAUDE.md 保持方向感。
Skill 承载可执行流程。
```

如果 CLAUDE.md 里出现大量 “第一步 / 第二步 / 第三步”，它大概率应该拆成 skill。

---

## 四、Plugins：把一组能力打包成可分发单元

Skill 解决的是“一段能力怎么复用”。Plugin 解决的是“一组能力怎么被别人安装和更新”。

【官方事实】Plugins 是包含组件的自包含目录，用来扩展 Claude Code。一个 plugin 可以包含 skills、agents、hooks、MCP servers、LSP servers、monitors、bin、settings 等。Plugin 的 manifest 可以位于 `.claude-plugin/plugin.json`，用于定义插件身份、描述、版本等元数据；如果组件都使用默认位置，manifest 本身可以省略。

推荐的团队分发结构：

```text
quality-tools/
├── .claude-plugin/
│   └── plugin.json
└── skills/
    └── quality-review/
        └── SKILL.md
```

`plugin.json`：

```json
{
  "name": "quality-tools",
  "description": "Quality review workflows for this team",
  "version": "1.0.0",
  "author": {
    "name": "Engineering Team"
  }
}
```

【作者推导】虽然 `plugin.json` 在简单场景下可选，但团队分发时建议保留。因为 name、description、version、author、userConfig、dependencies 等信息都需要稳定声明；没有 manifest 的 plugin 更适合本地试验或快速迁移。

【官方事实】`name` 是唯一标识，也会作为 skill namespace。例如 `quality-tools` 插件里的 `quality-review` skill 会以 `/quality-tools:quality-review` 形式出现。官方文档明确说，plugin skills 总是 namespaced，以避免多个插件有同名 skills 时冲突。

### 4.1 Plugin 能包含什么

【官方事实】Plugins reference 说明，plugin components 包括 skills、agents、hooks、MCP servers、LSP servers、monitors 等。Create plugins 文档还列出常见目录和字段：

| 目录 / 文件 / 字段 | 作用 |
| --- | --- |
| `.claude-plugin/plugin.json` | plugin manifest，可选但团队分发建议保留 |
| `skills/` | skills，推荐新插件使用 |
| `commands/` | 扁平 Markdown skills，兼容旧 custom commands |
| `agents/` | custom subagents |
| `hooks/hooks.json` | hooks 配置 |
| `.mcp.json` | MCP server 配置 |
| `.lsp.json` | LSP server 配置 |
| `output-styles/` | output style 配置 |
| `themes/` | color theme 配置 |
| `monitors/` | background monitors |
| `bin/` | 加入 Bash tool PATH 的可执行文件 |
| `settings.json` | 插件启用时应用的默认设置 |
| `userConfig` | 启用插件时向用户收集配置值 |
| `dependencies` | 声明依赖的其他 plugins |

> 注意：Plugins reference 中把 plugin monitors 和 plugin themes 标注为 experimental components。本文只描述截至 2026-05-20 的观察结果，不将其视为稳定行为保证。

【官方事实】官方文档特别提醒：不要把 `commands/`、`agents/`、`skills/`、`hooks/` 放进 `.claude-plugin/` 目录。`.claude-plugin/` 里只放 `plugin.json`；其他组件目录位于 plugin root。

【作者推导】这个结构体现了 plugin 的设计原则：

```text
.claude-plugin/ 是元数据。
plugin root 是能力本体。
```

把能力文件塞进 `.claude-plugin/`，就像把源码塞进 `package.json`，结构会混乱，也容易加载失败。

### 4.2 plugin namespace 解决冲突

如果两个项目都有 `/deploy` skill， standalone 配置里容易冲突。

Plugin 通过 namespace 解决：

```text
/frontend-tools:deploy
/backend-tools:deploy
/security-tools:review
```

【作者推导】namespace 不是形式主义，而是团队分发的基本条件。没有 namespace，插件越多，命令越容易互相覆盖。

这也解释了为什么个人阶段可以用 `.claude/skills/deploy`，团队分发时更适合 plugin：

```text
个人：/deploy
团队：/platform-tools:deploy
```

用户一看就知道能力来自哪个包。

### 4.3 plugin settings 不是万能配置

【官方事实】Plugins 可以包含 `settings.json`，但 Create plugins 文档说明，当前只有 `agent` 和 `subagentStatusLine` keys 被支持。设置 `agent` 可以激活插件里的某个 custom agent 作为主线程，并应用该 agent 的 system prompt、tool restrictions 和 model。

【作者推导】这意味着不要把 plugin 当成任意 settings 分发包。`agent` 不是一个轻量 UI 开关，而是会改变主线程行为边界的配置。若要治理 permissions、marketplace allowlist、managed policy，应该使用 settings / managed settings 本身，而不是假设 plugin 可以覆盖所有配置。

Plugin 适合打包能力：

- skills
- agents
- hooks
- MCP servers
- LSP servers
- bin scripts
- 少量 plugin-supported settings

组织治理适合放在：

- project `.claude/settings.json`
- managed settings
- marketplace restrictions

### 4.4 userConfig：插件参数不要硬编码

【官方事实】Plugins reference 提供 `userConfig` 字段，用来声明启用插件时需要用户填写的配置项。配置项可以是 `string`、`number`、`boolean`、`directory` 或 `file`。如果字段标记 `sensitive: true`，Claude Code 会遮蔽输入，并把值放到安全存储中，而不是普通 `settings.json`。

示意：

```json
{
  "userConfig": {
    "api_endpoint": {
      "type": "string",
      "title": "API endpoint",
      "description": "Internal API endpoint",
      "required": true
    },
    "api_token": {
      "type": "string",
      "title": "API token",
      "description": "API authentication token",
      "sensitive": true
    }
  }
}
```

【作者推导】这解决的是“同一个插件，不同用户、不同环境如何配置”的问题。团队插件不应该把真实 token、内网地址、个人路径直接写进 plugin 文件；插件只声明需要什么配置，具体值由用户或组织环境提供。

适合放进 `userConfig`：

- 内部 API 地址
- 用户本地工具路径
- 功能开关
- 非共享凭据引用

不适合硬编码进 plugin：

- 生产密钥
- 个人 access token
- 只对某台机器有效的绝对路径
- 可以识别内部系统的敏感细节

---

## 五、什么时候从 skill 升级为 plugin

不是所有 skill 都值得变成 plugin。

【作者推导】升级的信号通常有五个：

| 信号 | 说明 |
| --- | --- |
| 一个 skill 开始依赖脚本、模板、示例、MCP | 需要目录和依赖管理 |
| 多个 skills 总是一起使用 | 需要打包成一个能力集合 |
| 需要 custom agent 或 hooks 配合 | 已经超出单 skill |
| 要跨项目或跨团队共享 | 需要 namespace、版本、安装方式 |
| 需要被 marketplace 发现和更新 | 需要 plugin 化 |

不该升级的情况：

- 只是你个人的一个小提示
- 只在当前项目有意义
- 还没有稳定下来
- 没有测试过触发条件
- 没有 README 或使用说明

可以用这棵树判断：

```text
这是一段常驻事实吗？
  是 -> CLAUDE.md
  否 -> 它是可复用流程或知识吗？
    否 -> 直接 prompt
    是 -> 只有一个 SKILL.md 就够吗？
      是 -> Skill
      否 -> 是否要和 agents / hooks / MCP / scripts 一起分发？
        是 -> Plugin
        否 -> Skill + supporting files
```

再问：

```text
这个 plugin 是否要给团队或社区安装？
  否 -> 本地 --plugin-dir 或项目配置
  是 -> Marketplace
```

---

## 六、Marketplaces：把插件变成可发现、可安装、可治理的目录

【官方事实】Plugin marketplace 是一个包含 `.claude-plugin/marketplace.json` 的仓库或目录。`marketplace.json` 定义 marketplace 的 name、owner 和 plugins 列表；每个 plugin entry 至少需要 `name` 和 `source`。

最小 marketplace：

```json
{
  "name": "team-tools",
  "owner": {
    "name": "Platform Team"
  },
  "plugins": [
    {
      "name": "quality-tools",
      "source": "./plugins/quality-tools",
      "description": "Quality review workflows for this team"
    }
  ]
}
```

这里有两个容易混淆的名字：

```text
team-tools     -> marketplace name
quality-tools  -> plugin name
```

`source: "./plugins/quality-tools"` 这样的相对路径，解析基准是 marketplace root，也就是包含 `.claude-plugin/` 的目录，而不是 `.claude-plugin/` 目录本身。

安装示意：

```text
/plugin marketplace add ./my-marketplace
/plugin install quality-tools@team-tools
```

安装命令格式是：

```text
/plugin install <plugin-name>@<marketplace-name>
```

【官方事实】Marketplace 支持多种 plugin source，例如相对路径、GitHub repo、git URL、npm package 等。相对路径必须以 `./` 开头。

### 6.1 版本策略：version 和 commit SHA

【官方事实】plugin version 可以来自 `plugin.json`、marketplace entry 或 git commit SHA。官方文档说明，如果 `plugin.json` 设置了 `version`，用户只有在该字段变化时才会收到更新；如果省略 version，git source 的每个 commit 都会被视为新版本。`plugin.json` 中的 version 会优先于 marketplace entry 中的 version。

【作者推导】团队内部一般有两种策略：

| 策略 | 适合 |
| --- | --- |
| 明确 `version` | 稳定发布、需要 changelog、避免每个 commit 都推给用户 |
| 省略 `version` 用 commit SHA | 快速迭代、内部试点、小范围验证 |

最怕的是两边都写 version，但忘了同步。官方文档也提醒：如果 `plugin.json` 和 marketplace entry 都写了 version，`plugin.json` 会静默优先，可能掩盖 marketplace 里的版本。

### 6.2 Marketplace 治理：允许谁进入来源列表

【官方事实】Marketplace 文档说明，组织可以用 managed settings 中的 `strictKnownMarketplaces` 控制用户能添加哪些 marketplace。空数组表示完全禁止添加新 marketplace；列表可以 allow 特定 GitHub repo、URL、hostPattern、pathPattern 等。

这里要区分两个配置：

| 配置 | 解决什么问题 | 不解决什么问题 |
| --- | --- | --- |
| `strictKnownMarketplaces` | 限制用户能添加哪些 marketplace | 不会自动把 marketplace 注册给用户 |
| `extraKnownMarketplaces` | 自动注册已允许的 marketplace | 不负责限制用户还能添加什么 |

【作者推导】换句话说，allowlist 只是门禁，不是分发。团队如果希望用户打开 Claude Code 就能看到 approved marketplace，通常需要同时考虑：

```text
strictKnownMarketplaces -> 控制“能不能添加”
extraKnownMarketplaces  -> 控制“是否自动出现”
```

【作者推导】这才是团队标准化真正落地的地方：

```text
Plugin 让能力可安装。
Marketplace 让能力可发现。
Managed settings 让来源可治理。
```

一个团队可以这样分层：

| 层 | 责任 |
| --- | --- |
| plugin author | 写 plugin、版本、README、测试 |
| marketplace maintainer | 审核 plugin source、分类、版本 |
| platform / admin | 用 managed settings 限制 marketplace 来源 |
| project lead | 决定项目启用哪些 plugins |

【官方 GitHub】`anthropics/claude-plugins-official` 是 Anthropic 管理的官方插件目录。它的 README 也提醒用户：安装、更新或使用插件前必须信任插件，因为插件可能包含 MCP servers、文件或其他软件，Anthropic 无法保证每个第三方插件行为和安全。

【作者推导】这句话对企业内部同样成立：marketplace 不是“安全豁免区”。它只是让分发可控。真正的安全仍然来自 review、版本 pin、managed settings、permissions、hooks 和最小凭据。

---

## 七、机制解释：从“个人习惯”到“团队标准”的三个转换

### 7.1 从常驻 context 到按需 context

CLAUDE.md 会让规则常驻。Skill 让流程按需加载。Plugin 和 marketplace 让这种按需能力可以安装和升级。

【作者推导】这是 context 成本的转换：

```text
写进 CLAUDE.md：每次都付 token 成本。
写成 Skill：需要时才付成本。
打成 Plugin：别人也能安装同一套能力。
进入 Marketplace：团队能治理来源和版本。
```

这就是为什么“把所有规范写进 CLAUDE.md”不是长期答案。

### 7.2 从单文件维护到组件边界

Skill 主要是 `SKILL.md` 和支持文件。Plugin 则把能力拆成组件：

```text
skills/     -> 知识和流程
agents/     -> 角色和 context 隔离
hooks/      -> 确定性控制
.mcp.json   -> 外部系统连接
bin/        -> 可执行脚本
userConfig  -> 用户或环境差异
```

【作者推导】这个边界让维护责任更清楚：

- 流程变了，改 skill
- 检查逻辑变了，改 hook
- 外部系统变了，改 MCP
- 角色提示变了，改 agent
- 分发版本变了，改 plugin / marketplace

如果所有内容都塞进一个巨大 skill，后续团队维护会很痛。

### 7.3 从复制粘贴到版本治理

个人分享常常是“把这个目录复制给你”。团队标准化不能靠复制粘贴。

【作者推导】因为复制粘贴没有：

- 版本号
- 更新机制
- 来源可信
- namespace
- 撤回机制
- 安装范围
- 变更审计

Marketplace 解决的不是“怎么让别人拿到文件”，而是“怎么让别人以可治理的方式安装和更新”。

---

## 八、常见坑和修正

### 坑 1：把 CLAUDE.md 写成流程手册

如果 CLAUDE.md 里有大量流程步骤、长 checklist、示例和命令，它就会不断消耗 context。

修正：

- 常驻事实留在 CLAUDE.md
- 可复用流程拆成 skill
- 长 reference 放 skill supporting files

### 坑 2：skill 描述太泛

错误写法：

```yaml
description: Helps with code.
```

修正：

```yaml
description: Reviews uncommitted git changes for correctness, missing tests, accidental secrets, and risky deployment changes. Use before committing or opening a PR.
```

Skill 是否自动触发，很大程度取决于 description 是否精准。

### 坑 3：让 Claude 自动调用有副作用的 skill

发布、提交、发消息、跑迁移，不应该让 Claude 自己决定时机。

修正：

```yaml
disable-model-invocation: true
```

并配合 permissions、hooks 或 CI 审查。

### 坑 4：把 plugin 组件放进 `.claude-plugin/`

错误结构：

```text
my-plugin/
└── .claude-plugin/
    ├── plugin.json
    └── skills/
```

修正：

```text
my-plugin/
├── .claude-plugin/
│   └── plugin.json
└── skills/
```

`.claude-plugin/` 放 manifest，能力目录放 plugin root。

### 坑 5：忘记 namespace

Standalone skill 叫 `/deploy`，plugin skill 叫 `/plugin-name:deploy`。这是为了避免多个插件同名技能冲突。

修正：文档和培训材料中使用完整命令名，例如：

```text
/platform-tools:deploy
/quality-tools:review-diff
```

### 坑 6：plugin version 没有发布纪律

有些团队写了 `version: "1.0.0"`，之后每次修改 plugin 却不 bump version，用户永远拿不到更新。

修正：

- 稳定发布：每次 release bump version
- 快速试点：省略 version，用 commit SHA
- 不要在 plugin.json 和 marketplace entry 同时维护两个版本

### 坑 7：把 marketplace 当安全保证

Marketplace 只是目录和分发机制，不保证每个插件安全。

修正：

- 只添加可信 marketplace
- 用 managed settings 限制来源
- 对插件里的 hooks、MCP、bin 做 code review
- 对写外部系统的能力保持 permissions / hooks / CI gate

### 坑 8：把密钥和个人路径写进 plugin

错误写法：

```json
{
  "apiToken": "real-production-token",
  "toolPath": "C:/Users/someone/private-tool.exe"
}
```

修正：用 `userConfig` 声明配置项，把敏感字段标记为 `sensitive: true`，让具体值在安装或启用时由用户环境提供。

【作者推导】Plugin 是分发单元，不是秘密保险箱。凡是会因“复制给别人”而泄露、失效或绑定个人机器的内容，都不应该直接进入 plugin 仓库。

---

## 九、实践练习

【实践练习】下面的练习建议在测试仓库完成，不连接生产系统，不放真实密钥。每个练习至少保留一种可追溯产物：目录结构截图、配置文件 diff、`/context` 输出、`/plugin` 输出、命令行输出或最小复现仓库。

### 练习 1：把一段 CLAUDE.md 流程拆成 skill

**输入**：找一段已有 CLAUDE.md 中的流程，例如“提交前检查清单”，把它移到 `.claude/skills/review-diff/SKILL.md`，并写清 `description`。  
**预期观察**：CLAUDE.md 变短；skill 可以通过 `/review-diff` 调用，也可能在相关请求中自动触发。  
**验证标准**：保存 CLAUDE.md diff、skill 文件 diff、调用截图或会话输出。记录 `/context` 是否能观察到常驻内容减少。

### 练习 2：对比 reference skill 和 task skill

**输入**：创建两个 skill：一个 `api-conventions` 放接口规范；一个 `deploy-checklist` 放发布步骤，并给 `deploy-checklist` 加 `disable-model-invocation: true`。  
**预期观察**：接口规范类 skill 更适合自动触发；发布类 skill 需要用户手动 `/deploy-checklist`。设置 `disable-model-invocation` 后，发布类 skill 的 description 不再作为自动触发线索进入普通 context。  
**验证标准**：保存两个 `SKILL.md` 文件和调用记录。说明 reference / task 是内容分类，不是触发模式；真正改变调用方式的是 frontmatter。

### 练习 3：创建最小 plugin

**输入**：创建 `quality-tools/`，包含 `.claude-plugin/plugin.json` 和 `skills/quality-review/SKILL.md`，用 `claude --plugin-dir ./quality-tools` 测试。  
**预期观察**：skill 以 `/quality-tools:quality-review` 形式出现；`/help` 或 `/plugin` 能看到 plugin namespace。  
**验证标准**：保存插件目录结构、`plugin.json`、`SKILL.md`、`/plugin` 或 `/help` 输出。额外记录：如果去掉 `plugin.json` 但保留默认目录结构，Claude Code 是否还能识别该 plugin。

### 练习 4：给 plugin 添加第二个组件

**输入**：在最小 plugin 基础上添加一个只读 `agents/security-reviewer.md` 或一个 `hooks/hooks.json`。  
**预期观察**：plugin 不再只是 skill，而是包含多个组件；agent 出现在 `/agents`，hook 可在 `/hooks` 中看到。  
**验证标准**：保存目录结构、组件文件、`/agents` 或 `/hooks` 输出。说明这个能力为什么应该随 plugin 一起分发，而不是独立复制。

### 练习 5：创建本地 marketplace

**输入**：创建一个本地 `my-marketplace/.claude-plugin/marketplace.json`，把练习 3 的 plugin 作为相对路径 source 加进去，然后运行：

```text
/plugin marketplace add ./my-marketplace
/plugin install quality-tools@team-tools
```

**预期观察**：Claude Code 能识别 marketplace，并从中安装 plugin。  
**验证标准**：保存 `marketplace.json`、`/plugin marketplace` 输出、安装输出和插件调用结果。说明 `quality-tools@team-tools` 中哪一段是 plugin name，哪一段是 marketplace name，并确认相对路径是从 marketplace root 解析。

### 练习 6：设计团队分发策略

**输入**：写一份 markdown 方案，说明团队准备维护哪些 plugins、由谁 review、如何 version、哪些 marketplace 允许添加、是否需要 managed settings。  
**预期观察**：方案能区分 plugin author、marketplace maintainer、platform admin 和 project lead 的职责。  
**验证标准**：保存设计文档，必须包含版本策略、安全审查点、marketplace 来源控制和回滚策略。

---

## 实践复盘建议

做完练习后，建议按三类整理：

- 与官方文档一致：skill 是否按需加载？`disable-model-invocation` 是否阻止自动调用，并让 description 不进入普通 context？plugin skill 是否带 namespace？如果存在 `.claude-plugin/`，里面是否只放 manifest？marketplace 是否通过 `marketplace.json` 安装插件？
- 与预期不同：是 skill description 不够准确、plugin 目录结构放错、version 没有 bump、相对路径 source 解析失败、private repo 凭据不可用，还是 marketplace 被 managed settings 限制？
- 文档没有覆盖：保留目录结构、配置 diff、`/plugin` 输出、`/context` 对比或最小复现仓库，后续 Claude Code 版本升级后再复测。

这篇文章的核心结论可以压成一句话：

```text
Skills 把个人经验变成按需能力，
Plugins 把一组能力变成可安装单元，
Marketplaces 把插件变成可发现、可更新、可治理的团队标准。
```

---

## 参考资料

### 官方主参考

- [Skills](https://code.claude.com/docs/en/skills)
- [Create plugins](https://code.claude.com/docs/en/plugins)
- [Plugin marketplaces](https://code.claude.com/docs/en/plugin-marketplaces)
- [Plugins reference](https://code.claude.com/docs/en/plugins-reference)

### GitHub / 社区补充

- [anthropics/skills](https://github.com/anthropics/skills)
- [anthropics/claude-plugins-official](https://github.com/anthropics/claude-plugins-official)
- [subinium/awesome-claude-code](https://github.com/subinium/awesome-claude-code)

GitHub 和社区仓库更适合用来观察 skill 模板、插件目录结构、marketplace 分发方式和团队实践，不用于替代官方文档对 Claude Code 行为的定义。凡是社区经验与官方文档冲突，以官方文档和本地实践观察为准。
