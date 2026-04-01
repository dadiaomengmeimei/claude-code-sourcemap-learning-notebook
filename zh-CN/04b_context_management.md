# 第 4b 章：上下文管理 — 召回、压缩与渐进式披露

> **核心源码文件：** `context.ts`, `compact.ts`, `autoCompact.ts`, `microCompact.ts`, `snipCompact.ts`, `contextCollapse/`, `attachments.ts`, `findRelevantMemories.ts`, `toolSearch.ts`
>
> **阅读时间：** 40 分钟

---

## 学习目标

1. 理解 Claude Code 的**三阶段上下文生命周期**：注入 → 管理 → 压缩
2. 掌握 5 层压缩管线的端到端流程和设计取舍
3. 理解**渐进式披露**如何在 token 预算内最大化信息密度
4. 理解 Memory Prefetch 的异步召回机制

---

## 4b.1 上下文生命周期全景图

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    上下文生命周期                                         │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  Phase 1: 注入 (会话开始)                                                │
│  ├── getUserContext()  → CLAUDE.md + 当前日期                            │
│  ├── getSystemContext() → git status + branch + recent commits          │
│  ├── Skills listing → 可用 skill 的 frontmatter 摘要                    │
│  └── Memory prefetch → 异步预取相关记忆文件                              │
│                                                                         │
│  Phase 2: 渐进式披露 (运行时)                                            │
│  ├── ToolSearch → 延迟加载工具 schema                                    │
│  ├── Relevant Memory → 按需召回相关记忆                                  │
│  └── Dynamic Skills → 按需注入 skill 内容                                │
│                                                                         │
│  Phase 3: 压缩 (上下文膨胀时)                                            │
│  ├── toolResultBudget → 大结果持久化到磁盘                               │
│  ├── snipCompact → 裁剪旧工具结果                                        │
│  ├── microcompact → 压缩中间工具调用                                     │
│  ├── contextCollapse → 折叠已完成子任务                                   │
│  └── autocompact → 全量 LLM 摘要                                        │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 4b.2 Phase 1：上下文注入

### 4b.2.1 context.ts — 会话级上下文

每次会话开始时，Claude Code 收集两类上下文，且都使用 `memoize` 缓存（一次会话只计算一次）：

```typescript
// src/context.ts
export const getUserContext = memoize(async () => {
  // 1. 读取 CLAUDE.md 文件 (项目级 + 用户级 + .claude/rules/*.md)
  const claudeMd = getClaudeMds(filterInjectedMemoryFiles(await getMemoryFiles()));

  // 2. 当前日期
  const currentDate = `Today's date is ${getLocalISODate()}.`;

  return { claudeMd, currentDate };
});

export const getSystemContext = memoize(async () => {
  // Git 状态 (branch, status, recent commits)
  // 注意：CCR 模式和禁用 git 指令时会跳过
  const gitStatus = isEnvTruthy(process.env.CLAUDE_CODE_REMOTE)
    || !shouldIncludeGitInstructions()
    ? null
    : await getGitStatus();

  return { ...(gitStatus && { gitStatus }) };
});
```

**关键设计决策：**

| 决策 | 原因 |
|------|------|
| `memoize` 缓存 | 一次会话只计算一次，避免重复 I/O |
| git status 是快照 | 注释明确说 "this status is a snapshot in time" |
| compact 后清除缓存 | `runPostCompactCleanup()` 会 `getUserContext.cache.clear?.()` |
| CLAUDE.md 支持 `@import` | 可以引用外部文件，支持模块化配置 |

### 4b.2.2 CLAUDE.md 的多层级结构

```
CLAUDE.md 搜索路径（优先级从高到低）：
├── ~/.claude/CLAUDE.md              (用户级 - 全局偏好)
├── <project>/.claude/CLAUDE.md      (项目级 - 项目规则)
├── <project>/CLAUDE.md              (项目级 - 兼容旧格式)
├── <project>/.claude/rules/*.md     (项目级 - 模块化规则)
└── --add-dir 指定的额外目录          (显式附加)
```

**巧思：`--bare` 模式的语义**

```typescript
// --bare 意味着 "跳过我没要求的"，而不是 "忽略我要求的"
const shouldDisableClaudeMd =
  isEnvTruthy(process.env.CLAUDE_CODE_DISABLE_CLAUDE_MDS) ||
  (isBareMode() && getAdditionalDirectoriesForClaudeMd().length === 0);
```

`--bare` 跳过自动发现（cwd 目录遍历），但仍然尊重 `--add-dir` 显式指定的目录。

### 4b.2.3 git status 的精心截断

```typescript
// src/context.ts - getGitStatus()
const MAX_STATUS_CHARS = 2000;

const status = await execFileNoThrow(gitExe, ['status', '--short']);
const truncatedStatus = status.stdout.length > MAX_STATUS_CHARS
  ? status.stdout.slice(0, MAX_STATUS_CHARS) + '\n... (truncated)'
  : status.stdout;

return [
  `This is the git status at the start of the conversation.`,
  `Note that this status is a snapshot in time, and will not update.`,
  `Current branch: ${branch}`,
  `Main branch: ${mainBranch}`,
  `Status:\n${truncatedStatus || '(clean)'}`,
  `Recent commits:\n${log}`,
].join('\n\n');
```

---

## 4b.3 Phase 2：渐进式披露

渐进式披露的核心思想：**不要一次性把所有信息塞进上下文，而是按需加载。**

### 4b.3.1 ToolSearch — 工具 Schema 的延迟加载

当工具数量很多时（内置 40+ 加上 MCP 工具），全部发送 schema 会消耗大量 token。
ToolSearch 机制实现了工具的延迟加载：

```
初始请求发送给 API 的工具列表：
  tools: [
    Bash          (完整 schema ~500 tokens),
    Read          (完整 schema ~300 tokens),
    ToolSearch    (完整 schema ~200 tokens),     ← 搜索入口
    mcp__db__query  (defer_loading: true),       ← 只有名称，~5 tokens
    mcp__api__fetch (defer_loading: true),       ← 只有名称，~5 tokens
    ... 50+ 更多延迟工具 ...
  ]

  节省: 50 个工具 × ~400 tokens/工具 = ~20,000 tokens 节省
```

**延迟加载的判断逻辑：**

```typescript
// src/tools/ToolSearchTool/prompt.ts
export function isDeferredTool(tool: Tool): boolean {
  // 1. 显式 opt-out：alwaysLoad = true 的工具永远不延迟
  if (tool.alwaysLoad === true) return false;

  // 2. MCP 工具总是延迟（workflow-specific，用户可能有几十个）
  if (tool.isMcp === true) return true;

  // 3. ToolSearch 自身不能延迟（模型需要它来加载其他工具）
  if (tool.name === TOOL_SEARCH_TOOL_NAME) return false;

  // 4. Agent 工具在 fork 模式下不延迟（需要第一轮就可用）
  if (feature('FORK_SUBAGENT') && tool.name === AGENT_TOOL_NAME) {
    if (isForkSubagentEnabled()) return false;
  }

  // 5. 其他标记了 shouldDefer 的工具
  return tool.shouldDefer === true;
}
```

**自动模式（tst-auto）的 token 阈值：**

```typescript
// src/utils/toolSearch.ts
// 只有当延迟工具的总 token 数超过阈值时才启用 ToolSearch
// 默认阈值：上下文窗口的 N%（通过 GrowthBook 配置）
const deferredTokens = await getDeferredToolTokenCount(tools, ...);
if (deferredTokens !== null && deferredTokens < threshold) {
  return false; // 工具不多，不需要延迟加载
}
```

### 4b.3.2 Memory Prefetch — 异步记忆召回

Claude Code 有一个精巧的记忆召回系统，在每个用户轮次开始时异步预取相关记忆：

```typescript
// src/query.ts — 每个用户轮次触发一次
using pendingMemoryPrefetch = startRelevantMemoryPrefetch(
  state.messages,
  state.toolUseContext,
);
```

**完整的召回流程：**

```
用户输入 "修复 database 连接超时问题"
         │
         ▼
┌─────────────────────────────────────────────────┐
│ startRelevantMemoryPrefetch()                   │
│ ├── 提取最后一条用户消息的文本                    │
│ ├── 过滤：单词查询太短，跳过                      │
│ ├── 检查：已召回的记忆总字节数 < 限额             │
│ └── 启动异步 sideQuery（不阻塞主流程）            │
└─────────────────────────────────────────────────┘
         │ (异步，与主模型流式响应并行)
         ▼
┌─────────────────────────────────────────────────┐
│ findRelevantMemories()                          │
│ ├── scanMemoryFiles() → 扫描 ~/.claude/memory/  │
│ │   └── 读取每个 .md 的 frontmatter（前30行）    │
│ ├── formatMemoryManifest() → 构建文件清单         │
│ └── selectRelevantMemories() → Sonnet 选择       │
│     ├── 输入：用户查询 + 记忆清单 + 最近工具      │
│     ├── 输出：最多 5 个最相关的文件名              │
│     └── 规则：不选已使用工具的参考文档             │
│            （但选 warnings/gotchas）              │
└─────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────┐
│ 在工具执行完成后的 collect 点消费                  │
│ ├── settledAt !== null → 已完成，注入结果          │
│ └── settledAt === null → 未完成，下轮重试          │
│     （prefetch 永远不阻塞当前轮次）               │
└─────────────────────────────────────────────────┘
```

**关键设计：Disposable 模式**

```typescript
// MemoryPrefetch 实现了 Symbol.dispose
// query.ts 用 `using` 绑定，所以在所有 generator 退出路径上
// （return, throw, .return() 闭包）都会自动 abort 和记录遥测
export type MemoryPrefetch = {
  promise: Promise<Attachment[]>;
  settledAt: number | null;      // promise 完成时设置
  consumedOnIteration: number;   // collect 点消费时设置
  [Symbol.dispose](): void;      // 自动清理
};
```

### 4b.3.3 Skills 的渐进式注入

Skills 系统也采用渐进式披露：

```
初始注入：skill_listing attachment
  ├── 只包含 frontmatter 摘要（名称 + 描述）
  ├── 每个 skill ~50 tokens（vs 完整内容 ~2000 tokens）
  └── 模型看到列表后，通过 SkillTool 按需加载完整内容

按需加载：dynamic_skill attachment
  ├── 模型调用 SkillTool({ skill: "deploy-to-prod" })
  ├── 完整 skill 内容注入为 attachment
  └── 后续轮次可直接使用
```

---

## 4b.4 Phase 3：5 层压缩管线

当对话变长时，上下文会膨胀。Claude Code 有 5 层压缩策略，按成本从低到高排列：

```
执行顺序和成本：
toolResultBudget → snip → microcompact → collapse → autocompact
     便宜          便宜      中等          中等        昂贵
     本地          本地      本地/API      本地        API调用
     O(n)          O(n)      O(n)          O(n)        O($$)
```

### 4b.4.1 Layer 1: applyToolResultBudget — 大结果落盘

```typescript
// 当单个 tool_result 超过预算时，持久化到磁盘，替换为引用
// 在 microcompact 之前运行（MC 按 tool_use_id 操作，不检查内容）
messagesForQuery = await applyToolResultBudget(
  messagesForQuery,
  toolUseContext.contentReplacementState,
  persistReplacements ? records => void recordContentReplacement(...) : undefined,
  // 排除没有 maxResultSizeChars 限制的工具
  new Set(tools.filter(t => !Number.isFinite(t.maxResultSizeChars)).map(t => t.name)),
);
```

### 4b.4.2 Layer 2: snipCompact — 裁剪旧结果

```typescript
// feature: HISTORY_SNIP
// 保留消息结构（tool_use + tool_result 配对），但清空旧的 tool_result 内容
// 返回 tokensFreed 给 autocompact 参考
const snipResult = snipModule.snipCompactIfNeeded(messagesForQuery);
messagesForQuery = snipResult.messages;
snipTokensFreed = snipResult.tokensFreed;
```

### 4b.4.3 Layer 3: microcompact — 压缩中间工具调用

microcompact 有两种模式：

**时间触发模式：**
```typescript
// 如果距离上次 assistant 消息超过阈值（服务器缓存已过期），
// 全量清理旧 tool results（因为缓存已冷，前缀会被重写）
const timeBasedResult = maybeTimeBasedMicrocompact(messages, querySource);
```

**缓存编辑模式（Cached MC）：**
```typescript
// 利用 API 的 cache_edits 功能，在不失效缓存前缀的情况下删除 tool results
// 不修改本地消息内容，而是在 API 层添加 cache_reference 和 cache_edits
const cacheEdits = mod.createCacheEditsBlock(state, toolsToDelete);
```

**只压缩特定工具的结果：**
```typescript
const COMPACTABLE_TOOLS = new Set([
  FILE_READ_TOOL_NAME,    // 文件内容可以重新读取
  ...SHELL_TOOL_NAMES,    // 命令输出可以重新执行
  GREP_TOOL_NAME,         // 搜索结果可以重新搜索
  GLOB_TOOL_NAME,         // 文件列表可以重新列举
  WEB_SEARCH_TOOL_NAME,   // 搜索结果
  WEB_FETCH_TOOL_NAME,    // 网页内容
  FILE_EDIT_TOOL_NAME,    // 编辑确认
  FILE_WRITE_TOOL_NAME,   // 写入确认
]);
```

### 4b.4.4 Layer 4: contextCollapse — 子任务折叠

这是最优雅的设计 — **读时投影**而非写时修改：

```
REPL 消息数组 (完整历史，从不修改):
  [msg1, msg2, msg3, msg4, msg5, msg6, msg7, msg8]

Collapse Store (摘要存储):
  commit1: { archived: [msg1, msg2, msg3], summary: "..." }
  commit2: { archived: [msg4, msg5], summary: "..." }

projectView() 的输出 (每次迭代重新计算):
  [summary1, summary2, msg6, msg7, msg8]

优势：
  ✓ 原始历史完整保留（可回溯）
  ✓ 摘要跨轮次持久化
  ✓ 不同视图可以有不同的折叠策略
```

**为什么 collapse 在 autocompact 之前运行？**

```typescript
// 源码注释：
// Runs BEFORE autocompact so that if collapse gets us under the
// autocompact threshold, autocompact is a no-op and we keep granular
// context instead of a single summary.
```

折叠保留了更细粒度的上下文（每个子任务的独立摘要），而全量压缩会把所有东西压成一个大摘要。

### 4b.4.5 Layer 5: autocompact — 全量 LLM 摘要

最后手段，调用 Claude 生成整个对话的摘要：

```typescript
// src/services/compact/autoCompact.ts
export async function autoCompactIfNeeded(messages, toolUseContext, ...) {
  // 1. 环境变量禁用检查
  if (isEnvTruthy(process.env.DISABLE_COMPACT)) return { wasCompacted: false };

  // 2. 熔断器：连续失败 N 次后停止重试
  if (tracking?.consecutiveFailures >= MAX_CONSECUTIVE_AUTOCOMPACT_FAILURES) {
    return { wasCompacted: false };
  }

  // 3. 阈值检查
  const shouldCompact = await shouldAutoCompact(messages, model, querySource, snipTokensFreed);
  if (!shouldCompact) return { wasCompacted: false };

  // 4. 先尝试 Session Memory 压缩（实验性）
  const sessionMemoryResult = await trySessionMemoryCompaction(...);

  // 5. 调用 Claude 生成摘要
  const compactionResult = await compactConversation(messages, ...);

  // 6. 清理：重置 microcompact 状态、context collapse、CLAUDE.md 缓存等
  runPostCompactCleanup(querySource);

  return { wasCompacted: true, compactionResult };
}
```

**熔断器设计：**

```typescript
// 没有熔断器时，上下文不可恢复地超限的会话会在每轮都尝试压缩，
// 用注定失败的压缩请求轰炸 API
if (nextFailures >= MAX_CONSECUTIVE_AUTOCOMPACT_FAILURES) {
  logForDebugging(
    `autocompact: circuit breaker tripped after ${nextFailures} consecutive failures`
  );
}
```

### 4b.4.6 Reactive Compact — 被动压缩

除了主动的 autocompact，还有被动的 reactive compact，在 API 返回 413 (prompt too long) 时触发：

```typescript
// src/query.ts — 413 错误处理
if ((isWithheld413 || isWithheldMedia) && reactiveCompact) {
  const compacted = await reactiveCompact.tryReactiveCompact({
    hasAttempted: hasAttemptedReactiveCompact,  // 防止无限循环
    querySource,
    aborted: toolUseContext.abortController.signal.aborted,
    messages: messagesForQuery,
    cacheSafeParams: { ... },
  });

  if (compacted) {
    // 压缩成功，用压缩后的消息重试
    state = { ...state, messages: postCompactMessages, hasAttemptedReactiveCompact: true };
    continue;
  }
  // 压缩失败，报错退出（不进入 stop hooks，防止死循环）
}
```

**真实生产 bug 的教训：**

```typescript
// 源码注释：
// "Preserve the reactive compact guard — if compact already ran and
// couldn't recover from prompt-too-long, retrying after a stop-hook
// blocking error will produce the same result. Resetting to false
// here caused an infinite loop: compact → still too long → error →
// stop hook blocking → compact → … burning thousands of API calls."
```

---

## 4b.5 Post-Compact 清理

压缩后需要重置大量状态，这是一个容易出错的环节：

```typescript
// src/services/compact/postCompactCleanup.ts
export function runPostCompactCleanup(querySource?: QuerySource): void {
  // 只有主线程的压缩才重置主线程的模块级状态
  const isMainThreadCompact = querySource === undefined
    || querySource.startsWith('repl_main_thread')
    || querySource === 'sdk';

  // 1. 重置 microcompact 状态
  resetMicrocompactState();

  // 2. 重置 context collapse（如果是主线程）
  if (feature('CONTEXT_COLLAPSE') && isMainThreadCompact) {
    resetContextCollapse();
  }

  // 3. 清除 getUserContext 缓存（让 CLAUDE.md 重新加载）
  if (isMainThreadCompact) {
    getUserContext.cache.clear?.();
    resetGetMemoryFilesCache('compact');
  }

  // 4. 清除系统 prompt sections
  clearSystemPromptSections();

  // 5. 清除分类器审批缓存
  clearClassifierApprovals();

  // 6. 注意：故意不重置 sentSkillNames
  // 因为重新注入完整 skill_listing (~4K tokens) 是纯 cache_creation
}
```

**为什么子 Agent 不重置主线程状态？**

```typescript
// 子 Agent 在同一进程中运行，共享模块级状态
// 如果子 Agent 的压缩重置了 context collapse，
// 会破坏主线程的 committed log
const isMainThreadCompact = querySource?.startsWith('repl_main_thread');
```

---

## 4b.6 设计模式总结

### 可迁移的 5 个上下文管理模式

| # | 模式 | Claude Code 实现 | 通用应用 |
|---|------|-----------------|---------|
| 1 | **快照式注入** | `memoize` + 会话开始时收集 | 任何需要初始上下文的系统 |
| 2 | **渐进式披露** | ToolSearch + Memory Prefetch | 大型工具集、知识库系统 |
| 3 | **分层压缩** | 5 层管线，便宜优先 | 任何有 token 限制的 LLM 应用 |
| 4 | **读时投影** | Context Collapse 的 projectView | 需要保留完整历史的系统 |
| 5 | **熔断器** | autocompact 连续失败后停止 | 任何可能失败的自动化操作 |

### 与第 4 章的关系

本章是第 4 章 4.3 节"多层压缩策略"的**扩展和补充**：
- 第 4 章侧重于压缩管线在 query loop 中的**执行位置和时序**
- 本章侧重于上下文的**完整生命周期**：从注入到召回到压缩
- 新增了 ToolSearch 渐进式披露、Memory Prefetch 异步召回等第 4 章未覆盖的内容

---

## 本章小结

| 概念 | 关键文件 | 一句话总结 |
|------|----------|------------|
| 会话上下文 | `context.ts` | memoize 缓存的 CLAUDE.md + git status 快照 |
| 记忆召回 | `findRelevantMemories.ts` | Sonnet 选择最相关的 5 个记忆文件 |
| 异步预取 | `attachments.ts` | Disposable 模式，永不阻塞主流程 |
| 工具延迟加载 | `toolSearch.ts` | defer_loading 节省 ~20K tokens |
| 大结果落盘 | `toolResultBudget` | 超大 tool_result 持久化到磁盘 |
| 裁剪旧结果 | `snipCompact.ts` | 保留结构，清空内容 |
| 中间压缩 | `microCompact.ts` | 缓存编辑模式避免缓存失效 |
| 子任务折叠 | `contextCollapse/` | 读时投影，原始历史不修改 |
| 全量压缩 | `autoCompact.ts` | 最后手段，带熔断器 |
| 被动压缩 | `reactiveCompact` | 413 错误触发，防无限循环 |
| 压缩后清理 | `postCompactCleanup.ts` | 10+ 状态重置，区分主线程/子 Agent |

### 下一章预告
第 5 章将深入**多 Agent 系统**——Coordinator 如何编排多个子 Agent，
以及 Fork 模式如何实现 prompt cache 共享。
