好，我按模块逐一列出需要对齐的具体能力点。每个能力点标注了完整版 Claude Code 的行为描述和当前 rust-clw 的状态。

---

## 一、核心运行时（Query Runtime）

| # | 能力点 | Claude Code 行为 | rust-clw 现状 |
|---|--------|-----------------|--------------|
| 1.1 | 多轮对话主循环 | 模型输出 → 检测 tool_use → 执行工具 → 回灌 tool_result → 继续，直到无 tool 调用 | **已实现** |
| 1.2 | 流式输出 | 逐 token 流式返回，实时渲染到终端 | **已实现** |
| 1.3 | 自动 Compact | 当 token 用量超过阈值时自动压缩历史消息（autoCompact） | compact crate 有骨架，**未接入主循环** |
| 1.4 | Micro Compact | 对单条过长的 tool_result 做局部压缩，不压缩整个历史 | **无** |
| 1.5 | ContextTooLong 恢复 | API 返回上下文超长错误时，紧急执行 reactiveCompact 后重试 | 错误类型存在，**无恢复逻辑** |
| 1.6 | MaxOutputTokens 恢复 | 模型输出被截断时，追加 "请继续" 让模型接着输出 | **无** |
| 1.7 | Token Budget 控制 | 每轮计算 input_budget，决定是否压缩和压缩程度 | 字段存在，**未生效** |
| 1.8 | Stop Hooks | 每轮结束后执行注册的 hook，判断是否应该终止循环 | **无** |
| 1.9 | Memory Prefetch | 在首轮 query 前异步加载 CLAUDE.md 等记忆文件注入 system prompt | **无** |
| 1.10 | 中断与恢复 | 用户 Ctrl+C 中断当前生成，保留已有上下文可继续对话 | **无**（Ctrl+C 直接退出进程） |
| 1.11 | Usage 统计 | 每轮追踪 input_tokens / output_tokens / cache_read / cache_write | 部分有（累计 token），**无 cache 统计** |
| 1.12 | 多 Provider 流式协议统一 | Anthropic / OpenAI / Ollama 的流式事件统一映射到 StreamEvent | **已实现** |

---

## 二、工具层（Tools）

### 2.1 已有工具的完善

| # | 能力点 | Claude Code 行为 | rust-clw 现状 |
|---|--------|-----------------|--------------|
| 2.1.1 | Bash timeout | 支持 `timeout_ms` 参数，超时自动 kill | **已实现** |
| 2.1.2 | Bash 后台执行 | 长命令自动转后台 Task，不阻塞主循环 | **无**，全部同步阻塞 |
| 2.1.3 | Bash 输出截断 | 输出超过阈值时自动截断，保留头尾 | **无** |
| 2.1.4 | FileEdit replace_all | 支持 `replace_all` 参数批量替换所有匹配 | **无**，只有 `replacen(1)` |
| 2.1.5 | FileEdit 错误诊断 | 匹配不唯一时给出候选位置和上下文 | **无**，只返回 "not unique" |
| 2.1.6 | FileRead 大文件处理 | 超大文件自动截断并提示使用 offset/limit | **无** |
| 2.1.7 | Grep 输出格式 | 按文件分组、显示行号、支持上下文行（-A/-B/-C） | **无上下文行**，基础匹配 |

### 2.2 缺失工具

| # | 工具 | Claude Code 行为 | 优先级 |
|---|------|-----------------|--------|
| 2.2.1 | **WebFetch** | HTTP GET 获取 URL 内容，转成 markdown 可读文本 | P0 |
| 2.2.2 | **WebSearch** | 调用搜索引擎 API，返回摘要和链接 | P0 |
| 2.2.3 | **AgentTool** | 启动子 Agent（独立消息循环），执行委托任务后返回结果 | P0 |
| 2.2.4 | **TodoWrite** | 创建/更新结构化任务列表，支持状态流转（pending/in_progress/completed） | P1 |
| 2.2.5 | **NotebookEdit** | 编辑 Jupyter notebook 的指定 cell（按 index） | P1 |
| 2.2.6 | **MCPTool** | 调用 MCP server 暴露的工具 | P1 |
| 2.2.7 | **ListMcpResources** | 列出 MCP server 可用的资源 | P1 |
| 2.2.8 | **ReadMcpResource** | 读取指定 MCP 资源内容 | P1 |
| 2.2.9 | **TaskCreate** | 创建后台任务 | P1 |
| 2.2.10 | **TaskGet** | 查询后台任务状态和输出 | P1 |
| 2.2.11 | **TaskList** | 列出所有后台任务 | P1 |
| 2.2.12 | **TaskStop** | 停止指定后台任务 | P1 |
| 2.2.13 | **PowerShell** | Windows 下的 shell 执行工具 | P2 |
| 2.2.14 | **LSPTool** | 获取代码诊断信息（linter errors） | P2 |
| 2.2.15 | **SkillTool** | 读取并执行 Skill 文件 | P2 |

---

## 三、权限系统（Permissions）

| # | 能力点 | Claude Code 行为 | rust-clw 现状 |
|---|--------|-----------------|--------------|
| 3.1 | Auto 模式 | 只读工具自动通过，写入/执行自动通过 | **已实现** |
| 3.2 | Deny 模式 | 非只读工具全部拒绝 | **已实现** |
| 3.3 | Interactive 确认 | 弹出 Y/N 提示，用户决定是否允许 | **未实现**（直接报错） |
| 3.4 | Allow Once | 仅本次允许该操作 | **无** |
| 3.5 | Always Allow | 本次会话对该工具/路径永久允许 | **无** |
| 3.6 | 权限规则持久化 | 允许规则写入 `~/.claude/settings.json`，下次自动生效 | **无** |
| 3.7 | 权限判断收口 | 统一在 orchestrator 层做，工具内部不重复检查 | **有重复**，orchestrator 和工具内都做 |

---

## 四、上下文压缩（Compact）

| # | 能力点 | Claude Code 行为 | rust-clw 现状 |
|---|--------|-----------------|--------------|
| 4.1 | TruncateStrategy | 按消息从旧到新截断 | crate 中**有实现**，未接入 |
| 4.2 | SummaryStrategy | 调用模型对历史消息做摘要 | **无** |
| 4.3 | autoCompact | 基于 token 计数，自动触发压缩 | **无**（阈值字段存在但未检查） |
| 4.4 | microCompact | 对单条 tool_result 过长时局部压缩 | **无** |
| 4.5 | reactiveCompact | API 报错 context_too_long 后的紧急压缩 | **无** |
| 4.6 | Session Memory Compact | 压缩时提取关键信息写入 session memory | **无** |
| 4.7 | Context Collapse | 对长 tool_result 折叠为摘要 + 展开标记 | **无** |

---

## 五、后台任务系统（Tasks）

| # | 能力点 | Claude Code 行为 | rust-clw 现状 |
|---|--------|-----------------|--------------|
| 5.1 | TaskManager 生命周期 | 注册、运行、完成/失败状态机 | crate 中**有实现**，未接入 |
| 5.2 | LocalShellTask | 后台 shell 命令执行 | **无** |
| 5.3 | LocalAgentTask | 后台子 Agent 运行 | **无** |
| 5.4 | 通知回灌 | 任务完成后，结果作为消息注入主对话 | **无** |
| 5.5 | 任务可见性 | UI 中显示当前后台任务列表和状态 | **无** |
| 5.6 | 任务取消 | 可以停止运行中的后台任务 | trait 定义有，**无实际实现** |

---

## 六、MCP 集成

| # | 能力点 | Claude Code 行为 | rust-clw 现状 |
|---|--------|-----------------|--------------|
| 6.1 | MCP Client（stdio） | 通过 stdio 与 MCP server 进程通信 | **无**（仅 config struct 占位） |
| 6.2 | 配置加载 | 从 `~/.claude/mcp.json` 读取 server 配置 | **无** |
| 6.3 | 自动启动 server | 根据配置自动 spawn MCP server 进程 | **无** |
| 6.4 | 工具发现 | 调用 `tools/list` 获取远程工具定义 | **无** |
| 6.5 | 工具注册 | 将 MCP 工具注入统一 ToolRegistry | **无** |
| 6.6 | 工具调用 | 通过 MCP 协议调用远程工具并返回结果 | **无** |
| 6.7 | 资源浏览 | `resources/list` 和 `resources/read` | **无** |
| 6.8 | 连接管理 | 心跳、超时、自动重连 | **无** |
| 6.9 | 多 server | 同时连接多个 MCP server，工具合并 | **无** |

---

## 七、会话管理

| # | 能力点 | Claude Code 行为 | rust-clw 现状 |
|---|--------|-----------------|--------------|
| 7.1 | 会话持久化 | 消息历史自动保存到 `~/.claude/sessions/<id>.json` | **无**（退出即丢） |
| 7.2 | 会话恢复 | `--resume` / `--continue` 恢复上次会话 | **无** |
| 7.3 | 会话列表 | 可以查看历史会话并选择恢复 | **无** |
| 7.4 | Session Memory | 从对话中自动提取关键信息写入 `CLAUDE.md` | **无** |
| 7.5 | Session ID | 每次会话分配唯一 ID 用于追踪 | **有** SessionState.id |
| 7.6 | CWD 管理 | 记录并可切换工作目录 | **有** SessionState.cwd |

---

## 八、命令系统（Slash Commands）

| # | 能力点 | Claude Code 行为 | rust-clw 现状 |
|---|--------|-----------------|--------------|
| 8.1 | Command 框架 | `/command` 格式识别、分发、帮助系统 | **无** |
| 8.2 | `/help` | 列出所有可用命令 | **无** |
| 8.3 | `/compact` | 手动触发上下文压缩 | **无** |
| 8.4 | `/clear` | 清空当前对话 | **无** |
| 8.5 | `/cost` | 显示 token 用量和费用 | **无** |
| 8.6 | `/model` | 运行中切换模型 | **无** |
| 8.7 | `/permissions` | 查看/修改权限模式 | **无** |
| 8.8 | `/memory` | 查看/编辑 CLAUDE.md | **无** |
| 8.9 | `/exit` | 退出程序 | **无**（靠 exit/quit 文本匹配） |
| 8.10 | `/config` | 查看/修改配置 | **无** |
| 8.11 | `/resume` | 恢复历史会话 | **无** |
| 8.12 | `/diff` | 查看本次会话文件变更 | **无** |

---

## 九、REPL 交互体验

| # | 能力点 | Claude Code 行为 | rust-clw 现状 |
|---|--------|-----------------|--------------|
| 9.1 | 行编辑 | 光标移动、删除、Home/End | **无**（裸 stdin） |
| 9.2 | 历史回溯 | 上下箭头翻历史输入 | **无** |
| 9.3 | Tab 补全 | 斜杠命令、文件路径补全 | **无** |
| 9.4 | 多行输入 | 粘贴多行文本完整保留 | **无** |
| 9.5 | 流式 Markdown 渲染 | 代码块语法高亮、粗体斜体 | **无** |
| 9.6 | 工具执行 Spinner | 执行工具时显示进度动画和工具名 | **无** |
| 9.7 | Diff 渲染 | 文件编辑操作显示彩色 diff | **无** |
| 9.8 | Ctrl+C 中断 | 中断当前生成但不退出程序 | **无** |
| 9.9 | 状态栏 | 显示模型名、token 计数、会话信息 | **无** |
| 9.10 | 权限确认 UI | 弹出 Y/N/Always 选项让用户选择 | **无** |

---

## 十、Provider 与 API

| # | 能力点 | Claude Code 行为 | rust-clw 现状 |
|---|--------|-----------------|--------------|
| 10.1 | Anthropic 流式 | 完整支持 Messages API streaming | **已实现** |
| 10.2 | OpenAI 兼容 | 支持 OpenAI / Ollama 等 | **已实现** |
| 10.3 | 请求重试 | 429/5xx 自动退避重试 | **无** |
| 10.4 | Prompt Cache | 利用 Anthropic prompt caching 减少成本 | **无** |
| 10.5 | 模型 Fallback | 主模型失败时降级到备选模型 | **无** |
| 10.6 | Cost 计算 | 根据模型定价计算本次调用费用 | **无** |

---

## 汇总

| 模块 | 总能力点 | 已实现 | 部分/占位 | 未实现 |
|------|---------|--------|----------|--------|
| 核心运行时 | 12 | 3 | 1 | 8 |
| 工具（已有完善） | 7 | 1 | 0 | 6 |
| 工具（缺失） | 15 | 0 | 0 | 15 |
| 权限系统 | 7 | 2 | 0 | 5 |
| 上下文压缩 | 7 | 0 | 1 | 6 |
| 后台任务 | 6 | 0 | 2 | 4 |
| MCP | 9 | 0 | 1 | 8 |
| 会话管理 | 6 | 2 | 0 | 4 |
| 命令系统 | 12 | 0 | 0 | 12 |
| REPL 体验 | 10 | 0 | 0 | 10 |
| Provider | 6 | 2 | 0 | 4 |
| **合计** | **97** | **10** | **5** | **82** |

当前对齐度：**~15/97 ≈ 15%**（按能力点数），需要完成的有 **82 个具体能力点**。

这就是完整的对齐目标清单。要不要我基于这个细化的清单重新画甘特图？