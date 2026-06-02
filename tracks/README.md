# Tracks

`tracks/` 用来放成体系的学习方向。每个子目录都是一个长期可扩展的主题，而不是一次性文章集合。

## 当前 Tracks

| Track | 主题 | 说明 |
| --- | --- | --- |
| `claude-code-public-mechanisms/` | Claude Code 官方公开能力机制 | 从公开文档和官方能力出发，解释 Claude Code 的 context、工具、权限、hooks、MCP、多 Agent 和工程接入 |
| `agent-harness-engineering/` | Agent Harness 工程 | 基于 `shareAI-lab/learn-claude-code` 社区教学项目，理解 Claude Code-like harness 的通用工程骨架 |

## 新增 Track

新增方向时只需要新增一个目录：

```text
tracks/<topic-slug>/
├── README.md
├── 01-*.md
└── 02-*.md
```

不要为了新增方向重排顶层目录。

