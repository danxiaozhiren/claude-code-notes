# Agent Harness 实验手册

> 用途：为 `tracks/agent-harness-engineering/` 8 篇文章提供可复现实验题。
> 写法原则：每个实验按“输入 / 预期观察 / 验证标准 / 风险提示”组织。
> 来源边界：这些实验基于社区教学项目 `shareAI-lab/learn-claude-code` 和本仓库文章设计，不代表 Anthropic Claude Code 官方内部实现。

---

## 实验设计原则

- 实验优先放在临时目录或临时 clone 中运行。
- 不在真实项目根目录直接跑会写 `.tasks/`、`.mailboxes/`、`.worktrees/`、`.scheduled_tasks.json` 的教学代码。
- 涉及 shell、文件写入、worktree、部署类 MCP 工具时，先用只读命令或 mock server。
- 不把单次模型行为观察写成普遍保证。
- 记录实验时保留：输入、关键输出、观察结论、版本或日期。

## 风险分级

| 风险等级 | 适用实验 | 执行边界 |
| --- | --- | --- |
| 低 | AHE-001、AHE-002、AHE-003、AHE-005、AHE-007 | 只读文件、只记录日志、临时目录即可 |
| 中 | AHE-004、AHE-006、AHE-008、AHE-010、AHE-011、AHE-012、AHE-015 | 需要临时 clone 或 mock server，避免真实仓库和真实外部系统 |
| 高 | AHE-009、AHE-013、AHE-014、AHE-016 | 只在一次性测试仓库中做，真实部署、删除、分支清理、长 token 消耗全部禁用 |

建议实验目录：

```text
/tmp/agent-harness-lab/
```

如果需要运行 `learn-claude-code` 教学代码，建议临时 clone：

```text
cd /tmp/agent-harness-lab
git clone https://github.com/shareAI-lab/learn-claude-code.git
cd learn-claude-code
```

---

## 实验总表

| 编号 | 主题 | 关联文章 | 难度 | 建议产物 |
| --- | --- | --- | --- | --- |
| AHE-001 | 最小 Agent Loop | 1 | 入门 | loop 流程图 |
| AHE-002 | tool_use / tool_result 对齐 | 1、2 | 入门 | 一轮工具调用记录 |
| AHE-003 | 新增工具不改主循环 | 2 | 进阶 | 工具注册表对比 |
| AHE-004 | 权限拒绝回填 | 3 | 进阶 | 被拒绝的 tool_result |
| AHE-005 | PreToolUse / PostToolUse hook | 4 | 进阶 | hook 日志 |
| AHE-006 | Todo 与 Task 的边界 | 5、6 | 进阶 | Todo / Task 对照表 |
| AHE-007 | Skill 按需加载 | 5 | 进阶 | skill catalog 与加载记录 |
| AHE-008 | Compact 与 Memory 分工 | 5、6 | 进阶 | compact 前后对比 |
| AHE-009 | 错误恢复路径 | 6 | 高阶 | max_tokens / prompt too long 观察 |
| AHE-010 | 后台任务回流 | 6 | 进阶 | task_notification 记录 |
| AHE-011 | Cron 触发进入 loop | 6 | 进阶 | scheduled message 记录 |
| AHE-012 | Subagent 上下文隔离 | 7 | 进阶 | 主会话 / 子会话对比 |
| AHE-013 | Mailbox 与 Team Protocol | 7 | 高阶 | inbox 与 request_id 记录 |
| AHE-014 | Worktree 文件隔离 | 7 | 高阶 | `git status` / diff 对比 |
| AHE-015 | MCP 工具发现与命名 | 8 | 进阶 | `mcp__server__tool` 工具池记录 |
| AHE-016 | 完整 Harness 运行顺序 | 8 | 高阶 | 一轮完整 loop 复盘 |

---

## AHE-001：跑通最小 Agent Loop

- 关联文章：第 1 篇
- 输入：

```text
运行 s01_agent_loop/code.py，让模型回答一个需要简单工具或多轮观察的问题。
```

- 预期观察：

| 现象 | 说明 |
| --- | --- |
| 模型产生回复 | loop 至少能完成一次 LLM 调用 |
| 没有 tool_use 时结束 | stop 条件由模型是否继续请求工具决定 |
| messages 累积历史 | 下一轮模型能看到上一轮结果 |

- 验证标准：

```text
能画出 model -> response -> append messages -> stop 的最小流程。
```

- 风险提示：只在临时目录运行，不需要真实项目文件。

## AHE-002：观察 tool_use / tool_result 对齐

- 关联文章：第 1、2 篇
- 输入：

```text
让 agent 读取一个小文件，观察模型请求工具、handler 执行、tool_result 回填的过程。
```

- 预期观察：

| 现象 | 说明 |
| --- | --- |
| `tool_use_id` 出现 | 工具调用有唯一 ID |
| tool_result 带同一个 ID | 结果能匹配回请求 |
| 下一轮模型引用工具结果 | 外部观察进入模型上下文 |

- 验证标准：

```text
能指出哪一行是工具请求，哪一行是工具结果，二者如何关联。
```

- 风险提示：使用只读文件，避免让实验写入真实项目。

## AHE-003：新增工具不改主循环

- 关联文章：第 2 篇
- 输入：

```text
在教学代码里增加一个只读工具，例如 list_dir 或 word_count。
```

- 预期观察：

| 现象 | 说明 |
| --- | --- |
| 新工具只加入 TOOLS / TOOL_HANDLERS | 主循环不需要新增专用分支 |
| 模型能看到新 schema | 工具进入 tool pool |
| handler 返回结果后仍走 tool_result | 协议没有改变 |

- 验证标准：

```text
diff 中主 loop 基本不动，只新增工具定义和 handler。
```

- 风险提示：新增工具先做只读版本。

## AHE-004：权限拒绝也要回填

- 关联文章：第 3 篇
- 输入：

```text
配置 deny 规则，要求 agent 执行一个被拒绝的危险命令。
```

- 预期观察：

| 现象 | 说明 |
| --- | --- |
| handler 没执行 | 权限在执行前生效 |
| 返回 permission denied tool_result | 模型看到拒绝原因 |
| 模型尝试改用安全路径 | 拒绝结果参与下一轮推理 |

- 验证标准：

```text
不是静默失败；拒绝也进入 messages。
```

- 风险提示：危险命令用 mock 字符串或明显会被拦截的命令，不要真的执行破坏性操作。

## AHE-005：验证 hooks 生命周期

- 关联文章：第 4 篇
- 输入：

```text
注册 PreToolUse 和 PostToolUse hook，分别记录工具执行前后日志。
```

- 预期观察：

| 现象 | 说明 |
| --- | --- |
| PreToolUse 在 handler 前触发 | 可做权限、日志、审计 |
| PostToolUse 在 handler 后触发 | 可做输出检查、后处理 |
| 主 loop 不需要知道每个 hook 的业务 | 横切逻辑从 loop 分离 |

- 验证标准：

```text
日志顺序能证明 hook 的触发点。
```

- 风险提示：hook 先做只记录日志，不要直接运行 formatter 或外部服务。

## AHE-006：比较 Todo 和 Task

- 关联文章：第 5、6 篇
- 输入：

```text
用 todo_write 写当前步骤，再用 create_task 创建带 blockedBy 的任务图。
```

- 预期观察：

| 机制 | 观察 |
| --- | --- |
| Todo | 适合当前会话步骤，轻量、短生命周期 |
| Task | 写入 `.tasks/`，有 owner、status、blockedBy |

- 验证标准：

```text
能说明 Todo 是当前工作现场，Task 是可恢复任务对象。
```

- 风险提示：实验会写 `.tasks/`，放在临时目录。

## AHE-007：观察 Skill 按需加载

- 关联文章：第 5 篇
- 输入：

```text
准备两个 skill，只让模型执行其中一个适用任务。
```

- 预期观察：

| 现象 | 说明 |
| --- | --- |
| system prompt 只列 skill 名称和描述 | 常驻成本低 |
| 需要时调用 load_skill | 完整内容按需进入上下文 |
| 未用 skill 不加载正文 | 避免噪声和 token 浪费 |

- 验证标准：

```text
能区分 skill catalog 和 skill body。
```

- 风险提示：skill 描述要具体，否则模型可能不会主动加载。

## AHE-008：观察 Compact 与 Memory 分工

- 关联文章：第 5、6 篇
- 输入：

```text
构造较长对话，加入一条长期偏好到 memory，再触发 compact。
```

- 预期观察：

| 机制 | 观察 |
| --- | --- |
| Compact | 压缩活跃历史 |
| Memory | 长期事实在上下文之外保存 |
| System Prompt | 下一轮重新装配真实状态 |

- 验证标准：

```text
能说明哪些信息被压缩，哪些信息通过 memory 回来。
```

- 风险提示：不要把短期任务步骤写进长期 memory。

## AHE-009：观察错误恢复路径

- 关联文章：第 6 篇
- 输入：

```text
构造长输出或超长上下文，观察 max_tokens、continuation、reactive compact。
```

- 预期观察：

| 失败类型 | 恢复方式 |
| --- | --- |
| max_tokens | 升级输出预算或 continuation |
| prompt too long | reactive compact 后重试 |
| 临时错误 | retry / backoff |

- 验证标准：

```text
能说明失败被分类，而不是盲目 retry。
```

- 风险提示：避免消耗过多 API token，先用小规模实验；不要用真实长任务或真实大仓库制造上下文超限。

## AHE-010：后台任务完成后回流

- 关联文章：第 6 篇
- 输入：

```text
运行一个慢命令，例如 sleep 5 && echo done，并要求后台执行。
```

- 预期观察：

| 现象 | 说明 |
| --- | --- |
| 立即返回 background task ID | 主 loop 不阻塞 |
| 后续出现 task_notification | 异步结果回流 |
| 模型能基于通知继续 | 结果进入可见上下文 |

- 验证标准：

```text
后台任务不是静默执行，完成后必须产生 observation。
```

- 风险提示：使用 `sleep` 这类安全命令，不要跑真实部署。

## AHE-011：Cron 触发进入同一个 loop

- 关联文章：第 6 篇
- 输入：

```text
创建一个 1 分钟后触发的一次性 cron reminder。
```

- 预期观察：

| 现象 | 说明 |
| --- | --- |
| job 写入 `.scheduled_tasks.json` | 调度状态可持久化 |
| cron queue 收到触发 | scheduler 不直接调用模型 |
| `[Scheduled] ...` 注入 messages | 未来事件回到同一个 loop |

- 验证标准：

```text
定时任务不是模型记住了，而是 scheduler 生产事件。
```

- 风险提示：实验后清理临时 `.scheduled_tasks.json`。

## AHE-012：Subagent 上下文隔离

- 关联文章：第 7 篇
- 输入：

```text
让 subagent 调研一个目录并返回 summary，再让主 agent 继续其他任务。
```

- 预期观察：

| 现象 | 说明 |
| --- | --- |
| subagent 使用 fresh messages | 子任务上下文隔离 |
| 主会话只拿 summary | 中间过程不污染主上下文 |
| subagent 没有递归 task 工具 | 避免无限 delegation |

- 验证标准：

```text
能对比主会话 messages 和 subagent messages 的边界。
```

- 风险提示：subagent 工具权限仍需受控。

## AHE-013：Mailbox 与 Team Protocol

- 关联文章：第 7 篇
- 输入：

```text
生成两个 teammate，让其中一个 submit_plan，lead 再 review_plan。
```

- 预期观察：

| 现象 | 说明 |
| --- | --- |
| `.mailboxes/` 出现消息 | MessageBus 投递 |
| `request_id` 出现 | 请求响应可匹配 |
| plan_approval_response 回到 teammate | 协议状态驱动下一步 |

- 验证标准：

```text
能说明 mailbox 解决投递，protocol 解决语义。
```

- 风险提示：只在一次性测试仓库中运行；不要让 teammate 在审批前执行真实写操作。

## AHE-014：Worktree 文件隔离

- 关联文章：第 7 篇
- 输入：

```text
为两个任务分别创建 worktree，让不同 teammate 在各自 worktree 中写文件。
```

- 预期观察：

| 现象 | 说明 |
| --- | --- |
| `.worktrees/<name>` 出现 | 文件系统隔离 |
| task JSON 记录 worktree 字段 | 任务绑定工作区 |
| 主 checkout 不被直接修改 | 并行写入被隔离 |

- 验证标准：

```text
能用 git status / diff 证明改动发生在隔离 worktree。
```

- 风险提示：只在临时 clone 中运行；删除 worktree 前确认无未提交变更，不要清理真实项目分支。

## AHE-015：MCP 工具发现与命名

- 关联文章：第 8 篇
- 输入：

```text
连接 docs 和 deploy mock MCP server，观察工具池变化。
```

- 预期观察：

| 现象 | 说明 |
| --- | --- |
| `connect_mcp` 先发现工具 | 连接不是调用 |
| 出现 `mcp__docs__search` | 工具 namespace 生效 |
| MCP handler 返回 tool_result | 外部工具进入同一协议 |

- 验证标准：

```text
能说明 MCP 工具如何合并进 builtin tool pool。
```

- 风险提示：部署类工具只用 mock server 或只读状态查询，不要连接真实生产 API。

## AHE-016：复盘完整 Harness 一轮顺序

- 关联文章：第 8 篇
- 输入：

```text
在 s20 中组合 todo、MCP、background task、cron reminder，观察一轮 agent_loop。
```

- 预期观察：

| 顺序 | 观察 |
| --- | --- |
| 外部事件注入 | cron / background notification 先进入 messages |
| 上下文准备 | compact / memory / skills / MCP state 更新 |
| 模型调用 | 带当前 tool pool 调 LLM |
| 工具执行 | PreToolUse、handler、PostToolUse |
| 回填 | tool_result 回到下一轮 |

- 验证标准：

```text
能画出一轮完整 loop，并标出每个机制插入点。
```

- 风险提示：完整实验成本较高，先关闭真实外部系统，只用 mock；不要把调度、后台任务和 MCP 写入真实项目目录。
