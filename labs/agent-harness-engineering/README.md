# Agent Harness Engineering Labs

这个目录存放 Agent Harness 工程学习系列的实践练习。

## 文件

- [00-Agent-Harness-实验手册.md](00-Agent-Harness-实验手册.md)：围绕 agent loop、工具系统、权限、hooks、上下文工程、长任务、多 agent、MCP 和完整 harness 的实验题库。

## 使用方式

这些实验服务于 `tracks/agent-harness-engineering/` 的 8 篇文章。实验优先使用临时目录、临时 clone、mock server 和只读命令，避免在真实项目根目录直接运行会写文件、删 worktree、触发部署或连接生产系统的操作。

每个实验统一按下面结构组织：

```text
输入
预期观察
验证标准
风险提示
```

