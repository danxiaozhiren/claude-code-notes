# AI Agent 底层原理学习库

> 当前定位：个人 AI Agent / Claude Code 底层原理学习、技术文章写作与实践验证资料库。  
> 作者：wt  
> 当前仓库快照：2026-06-10。
> 旧 Claude Code 官方能力系列的文档核查日期以正文标注为准，复用或发布前需要重新核查。

## 项目目标

这个仓库用于沉淀个人对 AI Agent 底层原理的系统学习，不再只围绕单一文章系列组织，而是按“学习方向”长期扩展。

当前主线是：

```text
AI Agent 不是单个 prompt，
而是模型、工具、上下文、权限、记忆、调度和外部能力共同组成的 harness / runtime。
```

后续新增方向时，优先新增 `tracks/<topic>/`，不要重新搬动顶层目录。

## 资料边界

写作时优先把每个结论落到四类来源之一：

| 类型 | 说明 |
| --- | --- |
| 官方事实 | 来自官方文档、官方仓库或官方公开材料 |
| 社区教学实现 | 来自可信社区教学项目、开源 demo 或可运行样本 |
| 作者推导 | 基于官方事实、社区实现和工程经验形成的解释 |
| 动手观察 | 读者可以复现实验获得的现象 |

不要把社区教学实现写成官方内部实现，不要把旧版本行为直接写成当前事实。

## 目录结构

```text
.
├── README.md
├── tracks/
│   ├── README.md
│   ├── claude-code-public-mechanisms/
│   │   ├── README.md
│   │   └── 01-...08-*.md
│   └── agent-harness-engineering/
│       ├── README.md
│       └── 01-...08-*.md
├── labs/
│   ├── README.md
│   └── claude-code/
│       ├── README.md
│       └── 00-Claude-Code-实践练习题库.md
├── meta/
│   ├── README.md
│   └── claude-code-public-mechanisms/
│       ├── 00-Claude-Code-系统学习路线.md
│       └── 00-Claude-Code-八篇文章写作规划.md
└── references/
    └── README.md
```

## 当前学习方向

| Track | 定位 | 当前状态 |
| --- | --- | --- |
| `tracks/claude-code-public-mechanisms/` | 基于 Claude Code 官方公开能力，理解产品机制、使用边界和工程化实践 | 8 篇初稿已完成 |
| `tracks/agent-harness-engineering/` | 基于 `shareAI-lab/learn-claude-code` 社区教学项目，理解 Claude Code-like Agent Harness 通用工程机制 | 第 1-8 篇初稿已完成 |

## 推荐阅读顺序

1. 先读 `tracks/claude-code-public-mechanisms/`：建立 Claude Code 作为 agentic harness 的公开能力地图。
2. 再读 `tracks/agent-harness-engineering/`：用社区教学代码理解 agent loop、工具、权限、hooks、context、团队协作和 MCP 如何组合。
3. 配合 `labs/` 做实践练习，把机制理解落到可观察现象。
4. 用 `meta/` 里的规划文档复盘写作纪律、资料优先级和旧系列设计取舍。

## 新增学习方向规范

新增方向时使用这个结构：

```text
tracks/<topic-slug>/
├── README.md
├── 01-*.md
├── 02-*.md
└── ...
```

每个 track 的 `README.md` 至少说明：

- 学习对象
- 资料来源边界
- 文章列表
- 推荐阅读顺序
- 和其他 track 的关系

如果需要实验材料，放到：

```text
labs/<topic-slug>/
```

如果需要资料索引、术语表或来源政策，放到：

```text
references/
```

## 写作纪律

- 每篇文章头部必须标注核查日期。
- experimental、preview、research preview、beta 功能首次出现时必须标注状态。
- 不把社区经验包装成官方结论。
- 不把旧版本行为直接当作当前事实。
- 不把模型行为经验描述成强保证。
- 机制解释必须说明“为什么这样设计”，不能只复述 README 或文档。
- 实践练习按 `输入 / 预期观察 / 验证标准` 组织。
