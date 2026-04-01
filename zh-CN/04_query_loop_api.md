# 第4章：查询循环与 API 交互（深度版）

> **这是 Claude Code 的心脏。** `query.ts` (67KB, 1730行) 实现了从用户输入到 AI 响应的完整闭环。
> 本章将深入每一个工程巧思，让你理解为什么这样设计。

## 学习目标
1. 理解 query loop 的状态机设计（为什么用 `while(true)` + State 对象）
2. 理解 StreamingToolExecutor 的并发控制模型（sibling abort、progress wake-up）
3. 理解多层压缩策略（snip → microcompact → context collapse → autocompact）
4. 理解错误恢复的分层设计（escalate → multi-turn → reactive compact）
5. 理解 prompt cache 优化的全链路设计
6. 理解 token budget 的 diminishing returns 检测
7. **新增** 通过端到端请求追踪建立整体认知
8. **新增** 理解设计决策背后的方案对比（为什么不用其他方案）
9. **新增** 提炼可迁移到自己 Agent 系统中的设计模式

---

## 4.1 架构总览：为什么 query.ts 是一个状态机

### 4.1.1 核心设计决策：while(true) + State 对象

query.ts 的核心是一个 `while(true)` 循环，而不是递归调用。这是一个深思熟虑的设计：

```typescript
// 可变的跨迭代状态。循环体在每次迭代开始时解构它。
// Continue 站点写 `state = { ... }` 而不是 9 个单独的赋值。
type State = {
  messages: Message[]
  toolUseContext: ToolUseContext
  autoCompactTracking: AutoCompactTrackingState | undefined
  maxOutputTokensRecoveryCount: number
  hasAttemptedReactiveCompact: boolean
  maxOutputTokensOverride: number | undefined
  pendingToolUseSummary: Promise<ToolUseSummaryMessage | null> | undefined
  stopHookActive: boolean | undefined
  turnCount: number
  // 上一次迭代为什么继续。第一次迭代时为 undefined。
  // 让测试可以断言恢复路径是否触发，而无需检查消息内容。
  transition: Continue | undefined
}
```

** 巧思1：transition 字段**

注意 `transition` 字段 — 它记录了"为什么上一次迭代选择了 continue"。这不只是调试用的，
它还用于**防止恢复循环**：

```typescript
// 如果上一次已经做了 collapse_drain_retry，这次还是 413，
// 就不再尝试 drain，而是 fall through 到 reactive compact
if (state.transition?.reason !== 'collapse_drain_retry') {
  const drained = contextCollapse.recoverFromOverflow(...)
  // ...
}
```

**所有可能的 transition 原因**：

| transition.reason | 含义 | 触发条件 |
|---|---|---|
| `next_turn` | 正常的下一轮 | 工具执行完毕 |
| `max_output_tokens_recovery` | 输出被截断恢复 | stop_reason = max_output_tokens |
| `max_output_tokens_escalate` | 升级输出限制 | 从 8k 升级到 64k |
| `reactive_compact_retry` | 响应式压缩后重试 | prompt_too_long 错误 |
| `collapse_drain_retry` | 上下文折叠后重试 | prompt_too_long 错误 |
| `stop_hook_blocking` | Stop Hook 阻止停止 | Hook 返回 blocking error |
| `token_budget_continuation` | Token 预算未用完 | 还有预算继续 |

### 4.1.2 为什么不用递归？

早期版本可能是递归的（注释中还有 `query_recursive_call` checkpoint），但改成了循环：

```
递归的问题：
1. 每次递归创建新的闭包 → 内存泄漏（长对话可能有 100+ 轮）
2. 递归深度受限于调用栈
3. 状态传递需要大量参数
4. 无法在中间状态做 GC

while(true) + State 的优势：
1. 单一闭包，内存恒定
2. 无栈深度限制
3. 所有状态集中在 State 对象中
4. continue 站点可以精确控制哪些状态重置
```

** 巧思2：continue 站点的状态重置策略**

每个 continue 站点都精确控制哪些状态重置、哪些保留：

```typescript
// max_output_tokens 恢复：保留 hasAttemptedReactiveCompact
const next: State = {
  messages: [...messagesForQuery, ...assistantMessages, recoveryMessage],
  maxOutputTokensRecoveryCount: maxOutputTokensRecoveryCount + 1, // 递增
  hasAttemptedReactiveCompact, // ← 保留！不重置
  // ...
}

// 正常下一轮：重置恢复计数器
const next: State = {
  messages: [...messagesForQuery, ...assistantMessages, ...toolResults],
  maxOutputTokensRecoveryCount: 0, // ← 重置
  hasAttemptedReactiveCompact: false, // ← 重置
  // ...
}

// stop_hook_blocking：保留 hasAttemptedReactiveCompact
// 注释解释了为什么：
// "Preserve the reactive compact guard — if compact already ran and
// couldn't recover from prompt-too-long, retrying after a stop-hook
// blocking error will produce the same result. Resetting to false
// here caused an infinite loop: compact → still too long → error →
// stop hook blocking → compact → … burning thousands of API calls."
```

这个注释揭示了一个**真实的生产 bug**：重置 `hasAttemptedReactiveCompact` 导致了无限循环，
烧掉了数千次 API 调用。

### 4.1.3 完整的循环流程图

```
┌─────────────────────────────────────────────────────────────────────┐
│ query() 主循环 │
├─────────────────────────────────────────────────────────────────────┤
│ │
│ ┌──────────────────────────────────────────────────────────────┐ │
│ │ Phase 0: 预处理管线 (每次迭代) │ │
│ │ ├── applyToolResultBudget() → 大结果持久化到磁盘 │ │
│ │ ├── snipCompact() → 裁剪旧的工具结果 │ │
│ │ ├── microcompact() → 压缩中间工具调用 │ │
│ │ ├── contextCollapse() → 折叠已完成的子任务 │ │
│ │ └── autocompact() → 全量压缩（如果需要） │ │
│ └──────────────────────────────────────────────────────────────┘ │
│ │ │
│ ▼ │
│ ┌──────────────────────────────────────────────────────────────┐ │
│ │ Phase 1: API 调用 │ │
│ │ ├── prependUserContext() → 注入 CLAUDE.md 等 │ │
│ │ ├── callModel() 流式返回 → 逐块接收响应 │ │
│ │ ├── StreamingToolExecutor → 边接收边执行工具 │ │
│ │ └── 错误处理 → fallback model / withhold │ │
│ └──────────────────────────────────────────────────────────────┘ │
│ │ │
│ ┌─────────┴─────────┐ │
│ │ │ │
│ 有 tool_use? 无 tool_use │
│ │ │ │
│ ▼ ▼ │
│ ┌─────────────────────┐ ┌──────────────────────────────────┐ │
│ │ Phase 2: 工具执行 │ │ Phase 3: 停止判断 │ │
│ │ ├── 收集剩余结果 │ │ ├── 413 恢复 (collapse/compact) │ │
│ │ ├── 附件注入 │ │ ├── max_output_tokens 恢复 │ │
│ │ ├── 记忆预取消费 │ │ ├── Stop Hooks 执行 │ │
│ │ ├── 技能发现消费 │ │ ├── Token Budget 检查 │ │
│ │ └── 刷新工具列表 │ │ └── return Terminal │ │
│ └────────┬────────────┘ └──────────────────────────────────┘ │
│ │ │
│ ▼ │
│ ┌─────────────────────┐ │
│ │ Phase 4: 继续 │ │
│ │ state = next │ ──── continue ────→ 回到 Phase 0 │
│ └─────────────────────┘ │
│ │
└─────────────────────────────────────────────────────────────────────┘
```

### 4.1.4 端到端请求追踪：跟着一个请求走完全程

理解 query loop 最好的方式不是分模块看，而是**跟着一个真实请求走完全程**。

假设用户输入：`帮我修复 src/auth.ts 中的空指针 bug`

```
时间线                    发生了什么                                    数据变化
─────────────────────────────────────────────────────────────────────────────────────
T+0ms    用户按回车
         │
         ├── processUserInput()                                messages = [
         │   将用户输入包装为 Message                              { role: 'user',
         │   type: 'user', content: [{ type: 'text',               content: '帮我修复...' }
         │   text: '帮我修复 src/auth.ts 中的空指针 bug' }]       ]
         │
         ├── query() 入口
         │   ├── buildQueryConfig() → 快照不可变配置
         │   ├── startRelevantMemoryPrefetch() → 后台预取记忆
         │   └── 初始化 State 对象 (turnCount=0)
         │
         ▼ ═══ 进入 while(true) 循环 ═══

T+1ms    Phase 0: 预处理管线
         │
         ├── applyToolResultBudget()  → 无操作（第一轮没有工具结果）
         ├── snipCompact()            → 无操作（消息太少）
         ├── microcompact()           → 无操作
         ├── contextCollapse()        → 无操作
         └── autocompact()            → 无操作

T+2ms    Phase 1: 组装 API 请求
         │
         ├── prependUserContext()                                API payload = {
         │   注入 CLAUDE.md 内容到 system prompt                   system: [系统提示 + CLAUDE.md],
         │   注入 git status 等上下文                               tools: [Bash, Read, Edit, ...],
         │                                                         messages: [用户消息],
         ├── callModel()                                           model: 'claude-sonnet-4-...',
         │   发送 HTTP POST 到 Anthropic API                       max_tokens: 8192
         │   开始接收 SSE 流                                     }
         │
         ▼

T+500ms  Phase 1: 流式接收（持续 3-8 秒）
         │
         │  ┌─ SSE chunk 1: thinking block (如果启用 extended thinking)
         │  ├─ SSE chunk 2: text "让我先看看这个文件..."
         │  ├─ SSE chunk 3: tool_use 开始 { name: 'Read', id: 'tu_001' }
         │  │   └── StreamingToolExecutor.addTool('Read', 'tu_001')
         │  │       canExecuteTool(true) → 立即开始执行！
         │  │       Read('src/auth.ts') 在后台运行
         │  │
         │  ├─ SSE chunk 4: tool_use input 完成 { path: 'src/auth.ts' }
         │  ├─ SSE chunk 5: tool_use 开始 { name: 'Grep', id: 'tu_002' }
         │  │   └── StreamingToolExecutor.addTool('Grep', 'tu_002')
         │  │       canExecuteTool(true) → Read 是 ConcurrencySafe，并行！
         │  │       Grep('null.*pointer', 'src/') 在后台运行
         │  │
         │  └─ SSE chunk N: message_stop, stop_reason: 'tool_use'
         │
         ▼

T+4000ms Phase 2: 收集工具结果
         │
         ├── getRemainingResults()                               messages = [
         │   Read 已完成 → yield tool_result                       用户消息,
         │   Grep 已完成 → yield tool_result                       assistant(text + 2个tool_use),
         │                                                         user(2个tool_result)
         ├── 消费 memoryPrefetch（如果已完成）                   ]
         ├── 消费 skillDiscovery（如果有）
         └── needsFollowUp = true（有 tool_use）

         Phase 4: 继续
         state = { messages: [..., tool_results], turnCount: 1,
                   transition: { reason: 'next_turn' } }
         continue → 回到 Phase 0

         ▼ ═══ 第二次迭代 ═══

T+4001ms Phase 0: 预处理管线
         ├── snipCompact() → 可能裁剪第一轮的 Read 结果（如果文件很大）
         └── 其他 → 无操作

T+4002ms Phase 1: 第二次 API 调用
         │                                                       API payload = {
         │  messages 现在包含完整的对话历史                         messages: [
         │  前缀与上次相同 → prompt cache 命中！                      用户消息,        ← 缓存命中
         │  只需要为新增的 tool_result 付费                           assistant(...),   ← 缓存命中
         │                                                           user(tool_results) ← 新增
         │  Claude 看到文件内容，生成修复方案                       ]
         │  返回 tool_use: Edit('src/auth.ts', ...)               }
         │
         ▼

T+8000ms Phase 2: 执行 Edit
         │
         ├── Edit 不是 ConcurrencySafe → 独占执行
         ├── 权限检查 → 弹出确认对话框（如果需要）
         ├── 用户确认 → 执行编辑
         └── tool_result: { success: true }

         Phase 4: 继续 → 第三次迭代

         ▼ ═══ 第三次迭代 ═══

T+9000ms Phase 1: 第三次 API 调用
         │
         │  Claude 看到 Edit 成功
         │  返回 text: "已修复空指针 bug，添加了 null check..."
         │  stop_reason: 'end_turn'（没有 tool_use）
         │
         ▼

T+12000ms Phase 3: 停止判断
          │
          ├── 没有 413 错误 → 跳过
          ├── stop_reason != max_output_tokens → 跳过
          ├── handleStopHooks()
          │   ├── 执行用户定义的 stop hooks（如 lint 检查）
          │   ├── hooks 通过 → 继续停止流程
          │   └── 后台: extractMemories(), promptSuggestion()
          ├── checkTokenBudget() → budget 为 null → stop
          └── return { reason: 'completed' }

          ═══ query() 返回 ═══

总计: 3 次 API 调用，~12 秒，消耗 ~5000 input tokens + ~2000 output tokens
其中 prompt cache 节省了第 2、3 次调用的 ~80% input token 费用
```

**关键观察**：
- 第一次迭代的 Read 和 Grep **在 Claude 还在输出时就开始执行了**（StreamingToolExecutor 的价值）
- 第二次 API 调用的前缀与第一次完全相同 → **prompt cache 命中**（只付增量 token 的钱）
- Edit 工具**独占执行**（不与其他工具并行），因为它修改文件
- 整个过程是 3 次 `while(true)` 迭代，不是 3 次递归调用

---
## 4.2 StreamingToolExecutor — 流式并发执行引擎

### 4.2.1 核心问题：为什么需要流式执行？

传统方式：等 Claude 返回完整响应 → 解析所有 tool_use → 执行工具

```
传统方式 (串行):
 Claude 响应 [████████████████] 5s
 解析 tool_use 0.1s
 执行 Tool A [████] 2s
 执行 Tool B [██████] 3s
 执行 Tool C [███] 1.5s
 总计: 5 + 0.1 + 2 + 3 + 1.5 = 11.6s

流式方式 (StreamingToolExecutor):
 Claude 响应 [████████████████] 5s
 Tool A [████] ← 在 Claude 还在输出时就开始了！
 Tool B [██████] ← Tool A 完成前就开始了！
 Tool C [███] ← 并行执行！
 总计: 5 + max(2,3,1.5) = 8s 节省 31%
```

### 4.2.2 并发控制模型：为什么不用 Actor 或 DAG？

在深入实现之前，先理解**为什么选择这个方案**：

```
┌─────────────────────────────────────────────────────────────────────────┐
│ 方案对比：Agent 工具编排的三种范式                                       │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│ 方案 A: Actor 模型 (Erlang/Elixir 风格)                                │
│   每个工具一个 Actor，消息传递通信                                       │
│   优点: 天然隔离，故障不传播                                            │
│   缺点: JS 是单线程 → Actor 退化为 Promise                             │
│         progress 消息需要额外的 mailbox 机制                            │
│         sibling abort 需要 supervisor 树 → 过度设计                    │
│   结论: 在 JS 中 Actor 模型没有真正的并发优势，反而增加复杂度            │
│                                                                         │
│ 方案 B: DAG 调度 (LangGraph 风格)                                      │
│   预先定义工具之间的依赖关系，拓扑排序执行                               │
│   优点: 依赖关系显式，可以做最优调度                                    │
│   缺点: Claude 在运行时决定调用哪些工具 → 无法预先建 DAG               │
│         工具之间的依赖是隐式的（Bash A 的输出可能影响 Bash B）          │
│         需要等 Claude 完整响应才能建图 → 失去流式执行的优势             │
│   结论: DAG 适合预定义工作流，不适合 LLM 动态决策的场景                 │
│                                                                         │
│ 方案 C: 读写锁 + 流式添加 (Claude Code 的选择) ✓                       │
│   ConcurrencySafe = 读锁，非安全 = 写锁                                │
│   优点: 简单直觉（读可并行，写必独占）                                  │
│         支持流式添加（不需要等完整响应）                                 │
│         sibling abort 自然融入（共享 AbortController）                  │
│   缺点: 不能表达细粒度依赖（如 "Read A 必须在 Edit B 之前"）           │
│   结论: 在 LLM Agent 场景下，简单 > 精确                               │
│                                                                         │
│ 核心洞察: LLM 决定工具调用顺序，而 LLM 通常会按逻辑顺序输出            │
│ （先 Read 再 Edit），所以简单的读写锁就够了。                           │
│ 如果 LLM 输出了错误的顺序，Bash 错误的 sibling abort 会兜底。          │
└─────────────────────────────────────────────────────────────────────────┘
```

StreamingToolExecutor 使用了一个精巧的并发控制模型：

```typescript
// 核心判断：这个工具能否与其他工具并行执行？
private canExecuteTool(isConcurrencySafe: boolean): boolean {
  const executingTools = this.tools.filter(t => t.status === 'executing')
  return (
    executingTools.length === 0 || // 没有正在执行的工具
    (isConcurrencySafe && executingTools.every(t => t.isConcurrencySafe))
    // 或者：自己是安全的 AND 所有正在执行的也是安全的
  )
}
```

这实现了一个**读写锁**语义：

```
┌─────────────────────────────────────────────────────────┐
│ 并发安全矩阵 │
├─────────────────────────────────────────────────────────┤
│ │
│ 正在执行 \ 新工具 ConcurrencySafe NOT Safe │
│ ───────────────────────────────────────────── │
│ 无 立即执行 立即执行 │
│ ConcurrencySafe 并行执行 等待 │
│ NOT Safe 等待 等待 │
│ │
│ 类比： │
│ ConcurrencySafe = 读锁 (多个读可以并行) │
│ NOT Safe = 写锁 (写必须独占) │
│ │
└─────────────────────────────────────────────────────────┘
```

### 4.2.3 Sibling Abort — Bash 错误级联取消

这是一个非常精妙的设计。当多个 Bash 命令并行执行时，如果一个失败了，
其他的通常也没有意义了（因为 Bash 命令经常有隐式依赖链）：

```typescript
// 创建一个子 AbortController，不会影响父级
private siblingAbortController: AbortController

constructor(toolDefinitions, canUseTool, toolUseContext) {
  // siblingAbortController 是 toolUseContext.abortController 的子控制器
  // 取消它不会取消父级 → query loop 不会结束这个 turn
  this.siblingAbortController = createChildAbortController(
    toolUseContext.abortController,
  )
}

// 在工具执行中：
if (isErrorResult) {
  // 只有 Bash 错误才取消兄弟工具！
  // Read/WebFetch 等是独立的 — 一个失败不应该影响其他
  if (tool.block.name === BASH_TOOL_NAME) {
    this.hasErrored = true
    this.erroredToolDescription = this.getToolDescription(tool)
    this.siblingAbortController.abort('sibling_error')
  }
}
```

** 巧思3：三层 AbortController 层级**

```
toolUseContext.abortController (用户按 Ctrl+C)
 └── siblingAbortController (Bash 错误级联)
 ├── toolAbortController[0] (Tool A 的独立控制器)
 ├── toolAbortController[1] (Tool B 的独立控制器)
 └── toolAbortController[2] (Tool C 的独立控制器)

用户 Ctrl+C → 所有工具停止，query loop 结束
Bash A 失败 → B、C 停止，但 query loop 继续（收集错误结果）
权限拒绝 → 该工具停止，且冒泡到 query loop（结束 turn）
```

注意 toolAbortController 的 abort 事件监听器中的冒泡逻辑：

```typescript
toolAbortController.signal.addEventListener('abort', () => {
  // 如果不是 sibling_error，且父级没有 abort，且没有 discard
  // → 冒泡到父级（例如权限拒绝需要结束整个 turn）
  if (
    toolAbortController.signal.reason !== 'sibling_error' &&
    !this.toolUseContext.abortController.signal.aborted &&
    !this.discarded
  ) {
    this.toolUseContext.abortController.abort(
      toolAbortController.signal.reason,
    )
  }
}, { once: true })
```

### 4.2.4 Progress Wake-up 机制

工具执行过程中会产生 progress 消息（如 Bash 的实时输出）。
这些消息需要**立即**传递给 UI，而不是等工具完成：

```typescript
// Progress 消息存储在单独的队列中
type TrackedTool = {
  // ...
  results?: Message[] // 最终结果（等工具完成后才 yield）
  pendingProgress: Message[] // Progress 消息（立即 yield）
}

// 当有新的 progress 消息时，唤醒等待中的 getRemainingResults
if (update.message.type === 'progress') {
  tool.pendingProgress.push(update.message)
  // 唤醒！
  if (this.progressAvailableResolve) {
    this.progressAvailableResolve()
    this.progressAvailableResolve = undefined
  }
}

// getRemainingResults 中的等待逻辑：
async *getRemainingResults() {
  while (this.hasUnfinishedTools()) {
    // 先 yield 已完成的结果和 progress
    for (const result of this.getCompletedResults()) {
      yield result
    }
    // 如果还有执行中的工具，等待任一完成 OR progress 到达
    if (this.hasExecutingTools() && !this.hasCompletedResults()) {
      const progressPromise = new Promise<void>(resolve => {
        this.progressAvailableResolve = resolve // 注册唤醒回调
    })
    await Promise.race([...executingPromises, progressPromise])
  }
}
}
```

** 巧思4：Promise.race 实现非阻塞等待**

这是一个经典的"多信号等待"模式：
- 任何工具完成 → 唤醒 → yield 结果
- 任何 progress 到达 → 唤醒 → yield progress
- 两者都没有 → 继续等待

### 4.2.5 Streaming Fallback 处理

当 API 在流式传输过程中触发 fallback（切换到备用模型）时，
已经开始执行的工具需要被丢弃：

```typescript
if (streamingFallbackOccured) {
  // 1. 为已 yield 的 assistant 消息发送 tombstone（从 UI 和 transcript 中移除）
  for (const msg of assistantMessages) {
    yield { type: 'tombstone', message: msg }
  }

  // 2. 清空所有状态
  assistantMessages.length = 0
  toolResults.length = 0
  toolUseBlocks.length = 0
  needsFollowUp = false

  // 3. 丢弃旧的 executor，创建新的
  if (streamingToolExecutor) {
    streamingToolExecutor.discard() // 标记为 discarded
    streamingToolExecutor = new StreamingToolExecutor(...) // 全新的
  }
}
```

**为什么需要 tombstone？**

因为 partial 消息（特别是 thinking blocks）有无效的签名，
如果保留在消息历史中，会导致 API 返回 "thinking blocks cannot be modified" 错误。

---
## 4.3 多层压缩策略 — 上下文管理的瑞士军刀

Claude Code 有 **5 层**上下文压缩策略，按执行顺序排列：

```
┌─────────────────────────────────────────────────────────────┐
│ 压缩管线 (每次迭代) │
├─────────────────────────────────────────────────────────────┤
│ │
│ 1. applyToolResultBudget() │
│ └── 超大的 tool_result → 持久化到磁盘，替换为引用 │
│ └── 在 microcompact 之前运行（MC 按 tool_use_id 操作） │
│ │
│ 2. snipCompact() [feature: HISTORY_SNIP] │
│ └── 裁剪旧的工具结果（保留结构，删除内容） │
│ └── 返回 tokensFreed 给 autocompact 参考 │
│ │
│ 3. microcompact() │
│ └── 压缩中间的工具调用（保留首尾，压缩中间） │
│ └── 支持 cached microcompact（利用 API 缓存删除） │
│ │
│ 4. contextCollapse() [feature: CONTEXT_COLLAPSE] │
│ └── 折叠已完成的子任务为摘要 │
│ └── 在 autocompact 之前运行 → 如果折叠够了就不需要全量 │
│ └── 是"读时投影"而非"写时修改" │
│ │
│ 5. autocompact() │
│ └── 全量压缩：调用 Claude 生成摘要 │
│ └── 最后手段，成本最高 │
│ │
└─────────────────────────────────────────────────────────────┘
```

** 巧思5：执行顺序的精心设计**

```
toolResultBudget → snip → microcompact → collapse → autocompact
便宜 便宜 中等 中等 昂贵
本地 本地 本地/API 本地 API调用
```

便宜的先做，如果够了就不需要做昂贵的。特别是 contextCollapse 在 autocompact 之前：

```typescript
// 注释原文：
// Runs BEFORE autocompact so that if collapse gets us under the
// autocompact threshold, autocompact is a no-op and we keep granular
// context instead of a single summary.
```

折叠保留了更细粒度的上下文（每个子任务的摘要），而全量压缩会把所有东西压成一个大摘要。

### 4.3.1 Context Collapse 的"读时投影"设计

这是一个特别优雅的设计模式：

```typescript
// 注释原文：
// Nothing is yielded — the collapsed view is a read-time projection
// over the REPL's full history. Summary messages live in the collapse
// store, not the REPL array. This is what makes collapses persist
// across turns: projectView() replays the commit log on every entry.
```

```
REPL 消息数组 (完整历史，从不修改):
[msg1, msg2, msg3, msg4, msg5, msg6, msg7, msg8]

Collapse Store (摘要存储):
commit1: { archived: [msg1, msg2, msg3], summary: "..." }
commit2: { archived: [msg4, msg5], summary: "..." }

projectView() 的输出 (每次迭代重新计算):
[summary1, summary2, msg6, msg7, msg8]

优势：
- REPL 数组从不被修改 → 不会丢失原始数据
- 摘要可以随时重新生成
- 类似 Git 的 commit log → 可以"回放"折叠历史
```

---
## 4.4 错误恢复的分层设计

### 4.4.1 max_output_tokens 恢复（三阶段）

当 Claude 的输出被截断时，恢复策略分三个阶段：

```
阶段1: Escalate (升级限制)
  条件: 使用了默认的 8k 限制 && 没有用户覆盖
  动作: 重试同一请求，但 max_output_tokens = 64k
  特点: 无额外消息，无多轮开销

阶段2: Multi-turn Recovery (多轮恢复)
  条件: 升级后仍然被截断 || 已有用户覆盖
  动作: 注入恢复消息，让 Claude 继续
  消息: "Output token limit hit. Resume directly — no apology,
  no recap. Pick up mid-thought if that is where the cut
  happened. Break remaining work into smaller pieces."
  最多: 3 次

阶段3: 放弃
  条件: 3 次恢复都失败
  动作: 显示被截断的消息
```

** 巧思6：恢复消息的措辞**

注意恢复消息的精心措辞：
- "no apology" — 防止 Claude 浪费 token 道歉
- "no recap" — 防止 Claude 重复已说过的内容
- "Pick up mid-thought" — 允许从句子中间继续
- "Break remaining work into smaller pieces" — 引导 Claude 分块输出

### 4.4.2 prompt_too_long 恢复（三阶段）

```
阶段1: Context Collapse Drain
  条件: 有待提交的折叠 && 上次不是 collapse_drain_retry
  动作: 提交所有待处理的折叠
  成本: 零（纯本地操作）

阶段2: Reactive Compact
  条件: drain 不够 || 没有 collapse
  动作: 调用 Claude 生成摘要
  成本: 一次 API 调用
  特点: 只尝试一次（hasAttemptedReactiveCompact 守卫）

阶段3: 放弃
  条件: reactive compact 也不够
  动作: 显示错误消息
  关键: 不进入 Stop Hooks！
  原因: "hooks have nothing meaningful to evaluate. Running stop hooks
  on prompt-too-long creates a death spiral: error → hook blocking
  → retry → error → …"
```

### 4.4.3 Withhold 模式 — 延迟错误显示

这是一个关键的设计模式：在流式循环中，可恢复的错误被"扣留"而不是立即 yield：

```typescript
let withheld = false
if (contextCollapse?.isWithheldPromptTooLong(message, ...)) withheld = true
if (reactiveCompact?.isWithheldPromptTooLong(message)) withheld = true
if (reactiveCompact?.isWithheldMediaSizeError(message)) withheld = true
if (isWithheldMaxOutputTokens(message)) withheld = true
if (!withheld) yield yieldMessage // 只 yield 非扣留的消息
```

**为什么要 withhold？**

如果立即 yield 错误消息，SDK 消费者（如 Desktop App）会看到错误并终止会话。
但恢复循环还在运行 — 没人在听了。Withhold 让恢复有机会成功，
只有恢复失败时才显示错误。

** 巧思7：hoisted gate 防止 withhold/recover 不一致**

```typescript
// 在流式循环之前就确定 mediaRecoveryEnabled
const mediaRecoveryEnabled = reactiveCompact?.isReactiveCompactEnabled() ?? false

// 为什么？因为 CACHED_MAY_BE_STALE 可能在 5-30s 的流式传输中翻转
// 如果 withhold 时是 true，recover 时变成 false → 消息丢失！
```

---
## 4.5 Prompt Cache 全链路优化

### 4.5.1 缓存命中的条件

Anthropic API 的 prompt cache key 由以下部分组成：

```
cache_key = hash(
  system_prompt, ← 必须完全相同
  tools, ← 必须完全相同（包括顺序）
  model, ← 必须完全相同
  messages[0..N-1], ← 前缀必须完全相同（字节级）
  thinking_config, ← 必须完全相同
)
```

### 4.5.2 Claude Code 的缓存优化手段

**手段1：工具列表排序稳定**
```typescript
// tools.ts 中对工具排序，确保每次请求的工具定义前缀不变
// 即使 MCP 工具动态加载，排序也是确定性的
```

**手段2：不修改已发送的消息**
```typescript
// backfillObservableInput 只在 yield 的克隆上操作
// 原始 message 保持不变 → 下次请求的前缀字节相同
let yieldMessage: typeof message = message
if (addedFields) {
  clonedContent ??= [...message.message.content]
  clonedContent[i] = { ...block, input: inputCopy }
  yieldMessage = { ...message, message: { ...message.message, content: clonedContent } }
}
// 注释："The original `message` is left untouched for assistantMessages.push
// below — it flows back to the API and mutating it would break prompt caching
// (byte mismatch)."
```

**手段3：Fork 子 Agent 共享父级缓存**
```typescript
// CacheSafeParams — 必须与父级完全相同的参数
export type CacheSafeParams = {
  systemPrompt: SystemPrompt
  userContext: { [k: string]: string }
  systemContext: { [k: string]: string }
  toolUseContext: ToolUseContext
  forkContextMessages: Message[] // 父级的消息历史
}

// Fork 子 Agent 使用父级的消息作为前缀
// → 前缀完全相同 → 缓存命中
// → Fork 很便宜（只付增量 token 的钱）
```

**手段4：contentReplacementState 克隆**
```typescript
// 注释原文：
// Clone by default (not fresh): cache-sharing forks process parent
// messages containing parent tool_use_ids. A fresh state would see
// them as unseen and make divergent replacement decisions → wire
// prefix differs → cache miss.
```

**手段5：dumpPromptsFetch 单例**
```typescript
// 每次 query session 只创建一个 fetch wrapper
// 避免每次迭代创建新闭包 → 只保留最新的请求体 (~700KB)
// 而不是所有请求体 (~500MB for long sessions)
const dumpPromptsFetch = config.gates.isAnt
  ? createDumpPromptsFetch(toolUseContext.agentId ?? config.sessionId)
  : undefined
```

---
## 4.6 Token Budget — 智能的"继续"决策

Token Budget 是一个让 Claude 在单次 turn 中使用更多 token 的机制：

```typescript
export function checkTokenBudget(
  tracker: BudgetTracker,
  agentId: string | undefined,
  budget: number | null,
  globalTurnTokens: number,
): TokenBudgetDecision {
  // 子 Agent 不参与 budget（只有主线程）
  if (agentId || budget === null || budget <= 0) {
    return { action: 'stop', completionEvent: null }
  }

  const pct = Math.round((turnTokens / budget) * 100)
  const deltaSinceLastCheck = globalTurnTokens - tracker.lastGlobalTurnTokens

  // 巧思8：Diminishing Returns 检测
  const isDiminishing =
  tracker.continuationCount >= 3 && // 至少继续了 3 次
  deltaSinceLastCheck < DIMINISHING_THRESHOLD && // 这次增量 < 500 tokens
  tracker.lastDeltaTokens < DIMINISHING_THRESHOLD // 上次增量也 < 500

  // 如果没有 diminishing 且未达到 90% → 继续
  if (!isDiminishing && turnTokens < budget * COMPLETION_THRESHOLD) {
    return { action: 'continue', nudgeMessage: '...' }
  }

  // 否则停止
  return { action: 'stop', completionEvent: { diminishingReturns: isDiminishing } }
}
```

**为什么需要 Diminishing Returns 检测？**

```
场景：用户设置了 500k token 预算

没有 DR 检测：
  Turn 1: 50k tokens (有用的工作)
  Turn 2: 30k tokens (有用的工作)
  Turn 3: 20k tokens (有用的工作)
  Turn 4: 200 tokens (Claude 说 "I think we're done")
  Turn 5: 150 tokens (Claude 说 "Is there anything else?")
  Turn 6: 100 tokens (Claude 说 "Let me know if you need more")
  ... 继续浪费 token 直到 500k ...

有 DR 检测：
  Turn 1-3: 同上
  Turn 4: 200 tokens → delta < 500 → 记录
  Turn 5: 150 tokens → 连续两次 delta < 500 → 停止！
  节省了大量无意义的 token
```

---
## 4.7 消息队列与中断注入

### 4.7.1 Agent 作用域的消息队列

消息队列是进程全局的单例，但每个 Agent 只消费属于自己的消息：

```typescript
// 主线程只消费 agentId === undefined 的消息
// 子 Agent 只消费自己 agentId 的 task-notification
const queuedCommandsSnapshot = getCommandsByMaxPriority(
  sleepRan ? 'later' : 'next',
).filter(cmd => {
  if (isSlashCommand(cmd)) return false // slash 命令不在这里处理
  if (isMainThread) return cmd.agentId === undefined
  // 子 Agent 只消费 task-notification，不消费用户 prompt
  return cmd.mode === 'task-notification' && cmd.agentId === currentAgentId
})
```

** 巧思9：Sleep 工具触发更深的队列消费**

```typescript
const sleepRan = toolUseBlocks.some(b => b.name === SLEEP_TOOL_NAME)
// sleepRan ? 'later' : 'next'
```

当 Claude 主动调用 Sleep 工具时，它在等待某些异步任务完成。
此时消费 'later' 优先级的消息（包括其他 Agent 的完成通知），
而正常情况下只消费 'next' 优先级的消息。

### 4.7.2 Memory Prefetch — 异步预取与零等待消费

```typescript
// 在 query() 入口处启动预取（只触发一次）
using pendingMemoryPrefetch = startRelevantMemoryPrefetch(
  state.messages, state.toolUseContext,
)

// 在每次迭代的工具执行后尝试消费
if (
  pendingMemoryPrefetch &&
  pendingMemoryPrefetch.settledAt !== null && // 已完成
  pendingMemoryPrefetch.consumedOnIteration === -1 // 还没消费过
) {
  const memoryAttachments = filterDuplicateMemoryAttachments(
    await pendingMemoryPrefetch.promise,
    toolUseContext.readFileState, // 过滤已读过的文件
  )
  // ...
  pendingMemoryPrefetch.consumedOnIteration = turnCount - 1
}
```

**设计要点**：
- `using` 关键字确保在所有退出路径上清理（dispose）
- 预取在 model 流式传输期间并行运行
- 如果预取还没完成，跳过（零等待），下次迭代再试
- `readFileState` 过滤确保不会注入 Claude 已经读过的文件

---
## 4.8 Stop Hooks — 可编程的停止条件

Stop Hooks 不只是简单的"检查是否停止"，它是一个完整的可编程系统：

```
Claude 说 "我完成了"
 │
 ▼
handleStopHooks()
 │
 ├── 保存 CacheSafeParams (供 /btw 和 side_question 使用)
 │
 ├── 执行 Stop Hooks (用户定义的 shell 命令)
 │ ├── blocking error → 注入消息，继续循环
 │ ├── preventContinuation → 强制停止
 │ └── 通过 → 继续检查
 │
 ├── 如果是 Teammate:
 │ ├── 执行 TaskCompleted Hooks
 │ └── 执行 TeammateIdle Hooks
 │
 ├── 后台任务 (fire-and-forget):
 │ ├── executePromptSuggestion() → 生成后续问题建议
 │ ├── executeExtractMemories() → 提取记忆
 │ └── executeAutoDream() → 自动梦境（反思）
 │
 └── 清理:
 └── cleanupComputerUseAfterTurn() → 释放 CU 锁
```

** 巧思10：API 错误时跳过 Stop Hooks**

```typescript
// 如果最后一条消息是 API 错误，跳过 Stop Hooks
if (lastMessage?.isApiErrorMessage) {
 void executeStopFailureHooks(lastMessage, toolUseContext)
 return { reason: 'completed' }
}
```

为什么？因为 Stop Hooks 评估的是 Claude 的响应。如果 Claude 没有产生有效响应
（API 错误），Hooks 评估它会创建死亡螺旋：
error → hook blocking → retry → error → …

---
## 4.9 工具编排的两种模式

Claude Code 有两种工具编排模式，通过 feature gate 切换：

### 模式1：StreamingToolExecutor（新）
- 边接收 API 响应边执行工具
- 使用 addTool() 逐个添加
- 自动管理并发
- 支持 progress wake-up
- 支持 sibling abort

### 模式2：runTools（传统）
- 等 API 响应完成后批量执行
- 使用 partitionToolCalls() 分批

```typescript
// partitionToolCalls 的分批策略：
// 连续的 ConcurrencySafe 工具合并为一批并行执行
// 非安全工具单独一批串行执行

// 输入: [Glob, Grep, Read, Edit, Glob, Read]
// 分批: [[Glob, Grep, Read], [Edit], [Glob, Read]]
// ↑ 并行执行 ↑ 串行 ↑ 并行执行

function partitionToolCalls(toolUseMessages, toolUseContext): Batch[] {
  return toolUseMessages.reduce((acc, toolUse) => {
    const isConcurrencySafe = /* ... */
    // 如果当前工具安全 AND 上一批也是安全的 → 合并
    if (isConcurrencySafe && acc[acc.length - 1]?.isConcurrencySafe) {
      acc[acc.length - 1].blocks.push(toolUse)
    } else {
      acc.push({ isConcurrencySafe, blocks: [toolUse] })
    }
    return acc
  }, [])
}
```

**并发限制**：
```typescript
function getMaxToolUseConcurrency(): number {
  return parseInt(process.env.CLAUDE_CODE_MAX_TOOL_USE_CONCURRENCY || '', 10) || 10
}
```
默认最多 10 个工具并行，可通过环境变量覆盖。

---
## 4.10 QueryConfig — 不可变的配置快照

```typescript
// 在 query() 入口处快照一次，整个循环期间不变
const config = buildQueryConfig()

export type QueryConfig = {
  sessionId: SessionId
  gates: {
    streamingToolExecution: boolean // 是否启用流式工具执行
    emitToolUseSummaries: boolean // 是否生成工具使用摘要
    isAnt: boolean // 是否是 Anthropic 内部用户
    fastModeEnabled: boolean // 是否启用快速模式
  }
}
```

** 巧思11：为什么要快照？**

注释解释了：
```typescript
// Immutable values snapshotted once at query() entry. Separating these from
// the per-iteration State struct and the mutable ToolUseContext makes future
// step() extraction tractable — a pure reducer can take (state, event, config)
// where config is plain data.
```

这是为未来的架构重构做准备：
- State = 可变状态
- Config = 不可变配置
- Event = 输入事件
- → 纯函数 reducer: `(state, event, config) → newState`

这是 Redux/Elm 架构的影子，说明团队在考虑将 query loop 重构为更可测试的纯函数。

---
## 4.11 Tool Use Summary — 异步摘要生成

工具使用摘要是给移动端 UI 看的（移动端不显示完整的工具调用）：

```typescript
// 在工具执行完成后，异步生成摘要（不阻塞下一次 API 调用）
nextPendingToolUseSummary = generateToolUseSummary({
  tools: toolInfoForSummary,
  signal: toolUseContext.abortController.signal,
  isNonInteractiveSession: ...,
  lastAssistantText,
})
  .then(summary => summary ? createToolUseSummaryMessage(summary, toolUseIds) : null)
  .catch(() => null) // 失败静默

// 在下一次迭代开始时消费（Haiku ~1s，model streaming 5-30s）
if (pendingToolUseSummary) {
  const summary = await pendingToolUseSummary
  if (summary) yield summary
}
```

**设计要点**：
- 使用 Haiku 模型（快速、便宜）
- 异步执行，不阻塞主循环
- 只在主线程生成（子 Agent 不需要）
- 在下一次迭代消费（此时 Haiku 早已完成）

---
## 本章小结：工程巧思清单

| # | 巧思 | 解决的问题 |
|---|------|------------|
| 1 | transition 字段 | 防止恢复循环，让测试可断言 |
| 2 | continue 站点精确重置 | 防止无限循环（真实 bug） |
| 3 | 三层 AbortController | Bash 级联取消 vs 用户中断 vs 权限拒绝 |
| 4 | Promise.race 多信号等待 | Progress 实时传递 |
| 5 | 5 层压缩管线 | 便宜的先做，贵的后做 |
| 6 | 恢复消息措辞 | 防止 Claude 浪费 token |
| 7 | hoisted gate | 防止 withhold/recover 不一致 |
| 8 | Diminishing Returns | 防止浪费 token 在无意义的继续上 |
| 9 | Sleep 触发深队列消费 | 让等待中的 Agent 能收到通知 |
| 10 | API 错误跳过 Hooks | 防止死亡螺旋 |
| 11 | Config 快照 | 为纯函数 reducer 重构做准备 |

---
## 4.12 可迁移的设计模式 — 带走这些，用在你自己的 Agent 系统中

以上 11 个巧思是 Claude Code 特有的，但它们背后有**通用的设计模式**，
可以直接迁移到任何 Agent 系统中：

### 模式1：乐观恢复（Optimistic Recovery）

**在 Claude Code 中**：withhold 模式 — 扣留错误消息，尝试恢复，恢复失败再暴露。

**通用形式**：
```
收到错误 → 不立即传播 → 尝试恢复策略 A → 失败 → 尝试策略 B → 失败 → 传播错误
```

**适用场景**：
- API 网关的重试机制（502 不立即返回给客户端，先重试其他后端）
- 数据库连接池（连接断开不立即报错，先尝试重连）
- 分布式系统的 leader 选举（leader 失联不立即告知客户端，先选新 leader）

**关键约束**：withhold 和 recover 必须使用**同一个 gate 变量**（hoisted gate），
否则在 withhold 和 recover 之间条件翻转会导致消息丢失。

### 模式2：分层降级（Layered Degradation）

**在 Claude Code 中**：5 层压缩管线 — 便宜的先做，贵的后做。

**通用形式**：
```
问题发生 → 尝试零成本方案 → 不够 → 尝试低成本方案 → 不够 → 尝试高成本方案
```

**适用场景**：
- CDN 缓存策略（内存缓存 → 磁盘缓存 → 回源）
- 搜索引擎降级（精确匹配 → 模糊匹配 → 语义搜索）
- 负载均衡（本地实例 → 同区域 → 跨区域）

**关键约束**：每层的执行顺序必须固定，且前一层的结果要能被后一层感知
（如 snipCompact 返回 tokensFreed 给 autocompact 参考）。

### 模式3：状态机 + 转换记录（State Machine with Transition Log）

**在 Claude Code 中**：`while(true)` + `transition` 字段 — 记录为什么继续，防止恢复循环。

**通用形式**：
```
while (true) {
  state = process(state)
  state.transition = { reason, timestamp }
  if (shouldStop(state)) break
  if (isRecoveryLoop(state.transition, previousTransition)) break // 防止无限循环
}
```

**适用场景**：
- 工作流引擎（记录每步为什么跳转，防止循环审批）
- 游戏 AI 状态机（记录状态转换原因，防止行为抖动）
- 编译器优化 pass（记录每个 pass 的效果，防止优化循环）

**关键约束**：transition 记录不只是调试用的 — 它是**防止无限循环的守卫条件**。

### 模式4：读写锁语义的并发控制

**在 Claude Code 中**：ConcurrencySafe = 读锁，非安全 = 写锁。

**通用形式**：
```
操作分为两类：
- 只读操作（可并行）：多个只读操作可以同时执行
- 写操作（独占）：写操作执行时，其他所有操作等待
```

**适用场景**：
- 数据库的 MVCC（读不阻塞读，写阻塞一切）
- 文件系统的 flock（LOCK_SH vs LOCK_EX）
- Kubernetes 的 admission webhook（多个 mutating webhook 串行，validating 并行）

**关键约束**：在 LLM Agent 场景下，工具的读写属性是**静态声明**的（工具定义时确定），
不是运行时推断的。这比数据库简单得多，但也意味着不能表达细粒度依赖。

### 模式5：不可变配置快照（Immutable Config Snapshot）

**在 Claude Code 中**：`buildQueryConfig()` 在入口处快照一次，整个循环不变。

**通用形式**：
```
const config = snapshot(mutableGlobalConfig) // 入口处快照
for (const event of events) {
  state = reducer(state, event, config) // config 是不可变的
}
```

**适用场景**：
- React/Redux 的 store 设计（action → reducer → new state）
- 游戏引擎的帧更新（每帧开始时快照输入状态）
- 微服务的配置热更新（请求开始时快照配置，请求内不变）

**关键约束**：这是为**可测试性**服务的 — 纯函数 `(state, event, config) → newState`
比闭包捕获全局变量容易测试得多。

### 模式6：级联取消的层级控制（Hierarchical Cancellation）

**在 Claude Code 中**：三层 AbortController — 用户中断取消一切，Bash 错误只取消兄弟，权限拒绝冒泡到 turn。

**通用形式**：
```
Root AbortController (最高级别的取消)
  └── Group AbortController (组级别的取消)
    ├── Task A AbortController
    ├── Task B AbortController
    └── Task C AbortController

取消传播规则：
- Root 取消 → 所有 Group 和 Task 取消
- Group 取消 → 该组的所有 Task 取消，Root 不受影响
- Task 取消 → 根据原因决定是否冒泡到 Group
```

**适用场景**：
- 微服务的请求取消传播（gateway timeout → 取消所有下游调用）
- CI/CD 的 job 取消（取消 pipeline → 取消所有 stage → 取消所有 step）
- 浏览器的 fetch 取消（页面导航 → 取消所有进行中的请求）

**关键约束**：取消原因（`abort reason`）必须携带语义信息，
让子级能根据原因决定是否冒泡。不是所有取消都应该传播到父级。

---

### 下一章预告
第5章将深入 **多 Agent 系统**——理解 AgentTool 如何创建和管理子 Agent。

