# 核心运行时（Query Runtime）— 能力对齐分析

> 基于 `goal.md` 第一章 12 个能力点，逐一与 `crates/core/src/` 实际代码对比，并按重要性排序。

---

## 一、优先级排序

### P0 — 最关键（影响基本可用性）

| 优先级 | # | 能力点 | 为什么最优先 |
|--------|-----|--------|------------|
| 1 | 1.10 | **中断与恢复** | Ctrl+C 直接杀进程，用户无法安全中断生成，误操作即丢失整个会话 |
| 2 | 1.5 | **ContextTooLong 恢复** | 长对话必然触发上下文超限，无恢复 = 崩溃丢失全部上下文 |
| 3 | 1.3 | **自动 Compact** | 1.5 的主动防线——在超限前就压缩历史，骨架已有，接入成本相对低 |
| 4 | 1.6 | **MaxOutputTokens 恢复** | 输出截断意味着代码写一半、思路断一半，实现简单（追加 "请继续"） |

### P1 — 重要（影响生产力与成本）

| 优先级 | # | 能力点 | 为什么重要 |
|--------|-----|--------|----------|
| 5 | 1.7 | **Token Budget 控制** | 成本控制的基础，决定何时/如何压缩。没有它 autoCompact 无法正确触发 |
| 6 | 1.4 | **Micro Compact** | 单条 tool_result（如 grep 几千行）吃掉大量上下文窗口，局部压缩显著提升利用率 |
| 7 | 1.9 | **Memory Prefetch** | CLAUDE.md 加载是项目级上下文的基础，影响模型理解深度和首轮回答质量 |
| 8 | 1.11 | **Usage 统计** | 用户需要知道消耗 / 费用，也是优化 prompt caching 策略的前提 |

### P2 — 有价值但可延后

| 优先级 | # | 能力点 | 说明 |
|--------|-----|--------|------|
| 9 | 1.8 | **Stop Hooks** | 扩展性需求（自动提交、自动测试等），核心流程不依赖 |

### 已实现（无需排期）

| # | 能力点 |
|-----|--------|
| 1.1 | 多轮对话主循环 |
| 1.2 | 流式输出 |
| 1.12 | 多 Provider 流式协议统一 |

---

## 二、代码对齐检查

逐项对比 `goal.md` 描述与实际源码，检查是否准确。

### 1.1 多轮对话主循环 ✅ 一致

- **goal.md**: 已实现
- **代码**: `query.rs` 有完整的 `loop`，流程为 build request → stream → collect tool_use → execute → append tool_result → 循环

```rust
// crates/core/src/query.rs — 主循环骨架
loop {
    // ... build ModelRequest ...
    // ... stream response, collect tool_uses ...
    if tool_calls.is_empty() {
        return Ok(());
    }
    // ... execute tools, append results ...
    // loop back
}
```

### 1.2 流式输出 ✅ 一致

- **goal.md**: 已实现
- **代码**: `query.rs` 处理 `StreamEvent::TextDelta` 并 emit `QueryEvent::TextDelta`；`main.rs` 中 `handle_event_text` 逐 delta `print!` 到 stdout

### 1.3 自动 Compact ✅ 一致

- **goal.md**: compact crate 有骨架，未接入主循环
- **代码**:
  - `compact/src/strategy.rs` — `TruncateStrategy` 实现了 `CompactStrategy` trait
  - `compact/src/budget.rs` — `TokenBudget` 有 `should_compact()` 和 `input_budget()`
  - `core/src/session.rs` — `SessionConfig` 持有 `token_budget: TokenBudget`
  - **但** `query.rs` **零引用** compact crate，从未调用 `should_compact()`

### 1.4 Micro Compact ✅ 一致

- **goal.md**: 无
- **代码**: 全代码库无任何单条 tool_result 局部压缩逻辑

### 1.5 ContextTooLong 恢复 ✅ 一致

- **goal.md**: 错误类型存在，无恢复逻辑
- **代码**:
  - `error.rs` 定义了 `AgentError::ContextTooLong`
  - `query.rs` 中 stream 错误统一走 `return Err(AgentError::Provider(e))`，无模式匹配 context_too_long、无重试

### 1.6 MaxOutputTokens 恢复 ✅ 一致

- **goal.md**: 无
- **代码**:
  - `StopReason::MaxTokens` 枚举已定义（`response.rs`）
  - `query.rs` 在无 tool_call 时直接 `return Ok(())`，**未检查** stop_reason 是否为 MaxTokens

### 1.7 Token Budget 控制 ✅ 一致

- **goal.md**: 字段存在，未生效
- **代码**:
  - `TokenBudget` 有完整逻辑：`input_budget()`、`should_compact()`、`compact_threshold`
  - `query.rs` 仅使用 `session.config.token_budget.max_output_tokens` 填充请求
  - `should_compact()` 和 `input_budget()` **从未在 query 循环中被调用**

### 1.8 Stop Hooks ✅ 一致

- **goal.md**: 无
- **代码**: 无 hook 注册/执行机制，循环退出仅看 `tool_calls.is_empty()` 和 `max_turns`

### 1.9 Memory Prefetch ✅ 一致

- **goal.md**: 无
- **代码**: system prompt 来自 CLI `--system` 参数的静态字符串，无 CLAUDE.md 文件加载逻辑

### 1.10 中断与恢复 ✅ 一致

- **goal.md**: 无（Ctrl+C 直接退出进程）
- **代码**:
  - `main.rs` 无 `ctrlc` / `tokio::signal` 处理
  - `AgentError::Aborted` 已定义但**从未被触发**
  - Ctrl+C 确实直接杀进程

### 1.11 Usage 统计 ⚠️ 基本一致，可更精确

- **goal.md**: 部分有（累计 token），无 cache 统计
- **代码**:
  - `SessionState` 有 `total_input_tokens` / `total_output_tokens`，在 `MessageDone` 时累加
  - `Usage` struct **已定义** `cache_creation_input_tokens` 和 `cache_read_input_tokens` 字段（`Option<usize>`）
  - 但 `query.rs` 累加逻辑**忽略**了这两个字段，从未读取/展示
- **建议修正**: "cache 字段已定义但未读取/累加/展示"（比"无 cache 统计"更准确）

### 1.12 多 Provider 流式协议统一 ✅ 一致

- **goal.md**: 已实现
- **代码**: `ModelProvider` trait + `StreamEvent` 统一枚举，`anthropic.rs` 和 `openai_compat.rs` 各自实现映射

---

## 三、额外发现（goal.md 未提及）

### 3.1 每轮重建 PermissionPolicy

`query.rs:173-178` 每次循环都 `new` 一个新的 `RuleBasedPolicy`：

```rust
let tool_ctx = ToolContext {
    cwd: session.cwd.clone(),
    permissions: Arc::new(claw_permissions::RuleBasedPolicy::new(
        session.config.permission_mode,
    )),
    session_id: session.id.clone(),
};
```

这意味着即使未来实现 "Allow Once"（3.4）或 "Always Allow"（3.5），策略状态也会在每轮被丢弃。应改为 session 级别持有单一 Policy 实例。

### 3.2 Stream 错误无分类处理

`query.rs:128-131` 对所有 stream 错误统一 `return Err`：

```rust
Err(e) => {
    warn!(error = %e, "stream error");
    return Err(AgentError::Provider(e));
}
```

没有区分：
- **429** — 限流，应退避重试（对应 10.3 请求重试）
- **5xx** — 服务端错误，应重试
- **context_too_long** — 应触发 reactiveCompact（对应 1.5）

这是 1.5（ContextTooLong 恢复）和 10.3（请求重试）两个能力点的**共同前提**。

---

## 四、建议实施路径

```
Phase 1: 对话生存保障
  1.10 中断与恢复  ──→  1.5 ContextTooLong 恢复  ──→  1.3 自动 Compact
       │                       │
       │                       └── 前提: stream 错误分类（额外发现 3.2）
       └── 前提: tokio::signal + CancellationToken

Phase 2: 输出完整性
  1.6 MaxOutputTokens 恢复（独立，仅需检查 StopReason）

Phase 3: 成本与质量
  1.7 Token Budget 控制  ──→  1.4 Micro Compact
       │
       └── 前提: 1.3 已接入

Phase 4: 上下文质量
  1.9 Memory Prefetch  ──→  1.11 Usage 统计完善

Phase 5: 扩展性
  1.8 Stop Hooks
```

关键依赖：
- **1.3 依赖 1.7** — 没有 budget 计算，compact 触发时机是盲的
- **1.5 依赖 3.2（错误分类）** — 必须先能识别 context_too_long 错误
- **1.4 依赖 1.3** — micro compact 是 auto compact 的细粒度补充
