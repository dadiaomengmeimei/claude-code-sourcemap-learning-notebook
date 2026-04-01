# 第5章：多 Agent 系统（深度版）

> **AgentTool 是 Claude Code 中最复杂的单个文件（228KB）。** 它实现了一个完整的
> 多 Agent 运行时：进程内 fork、git worktree 隔离、tmux 团队协作、异步生命周期管理。
> 本章将深入每一个工程巧思。

## 学习目标
1. 理解 Agent 上下文隔离的“默认隔离，显式共享”设计
2. 理解 Fork 子 Agent 的 prompt cache 共享机制
3. 理解 runAgent 的 12 步清理流程（为什么每一步都不能少）
4. 理解权限模式的继承与覆盖规则
5. 理解异步 Agent 的生命周期管理（spawn → progress → complete/kill）
6. 理解工具集解析的多层过滤策略
7. **新增** 通过完整的 Coordinator 编排案例建立实战认知
8. **新增** 理解 Worker 间的通信机制和消息时序
9. **新增** 提炼可迁移到自己 Agent 系统中的设计模式
---

## 5.1 架构总览

### 5.1.1 Agent 系统的三种运行模式

```
┌─────────────────────────────────────────────────────────────────┐
│ Agent 运行模式 │
├─────────────────────────────────────────────────────────────────┤
│ │
│ 模式1: 同步子 Agent (默认) │
│ ┌──────────┐ ┌──────────┐ │
│ │ 主 Agent │────▶│ 子 Agent │ 共享 abortController │
│ │ (等待) │◀────│ (执行) │ 共享 setAppState │
│ └──────────┘ └──────────┘ 可以弹权限对话框 │
│ │
│ 模式2: 异步后台 Agent │
│ ┌──────────┐ ┌──────────┐ │
│ │ 主 Agent │ │ 后台 Agent│ 独立 abortController │
│ │ (继续工作)│ │ (独立执行)│ setAppState = no-op │
│ └──────────┘ └──────────┘ 不能弹权限对话框 │
│ │
│ 模式3: Fork 子 Agent │
│ ┌──────────┐ ┌──────────┐ │
│ │ 主 Agent │────▶│ Fork │ 继承父级全部工具 │
│ │ (等待) │◀────│ (执行) │ 继承 thinkingConfig │
│ └──────────┘ └──────────┘ 共享 prompt cache │
│ │
└─────────────────────────────────────────────────────────────────┘
```

### 5.1.2 关键文件地图

```
AgentTool/
├── AgentTool.tsx (228KB) ← 主入口，输入验证，调度逻辑
├── runAgent.ts (35KB) ← Agent 生命周期管理
├── agentToolUtils.ts (22KB) ← 工具集解析，结果处理，异步生命周期
├── loadAgentsDir.ts ← Agent 定义加载
├── prompt.ts (16KB) ← Agent 调度的 prompt
├── constants.ts ← 工具名常量
└── built-in/
  ├── generalPurposeAgent.ts ← 通用 Agent
  └── exploreAgent.ts ← 探索 Agent（只读）
```

---
## 5.2 createSubagentContext — "默认隔离，显式共享"

这是整个 Agent 系统最核心的设计决策。`createSubagentContext` 创建子 Agent 的执行上下文，
**默认隔离所有可变状态**，调用者必须显式 opt-in 共享：

```typescript
export function createSubagentContext(
  parentContext: ToolUseContext,
  overrides?: SubagentContextOverrides,
): ToolUseContext {
  return {
    // ===== 默认隔离的状态 =====

    // 文件缓存：克隆（不是共享）
    readFileState: cloneFileStateCache(
      overrides?.readFileState ?? parentContext.readFileState,
    ),

    // 全新的集合（不继承父级的）
    nestedMemoryAttachmentTriggers: new Set<string>(),
    loadedNestedMemoryPaths: new Set<string>(),
    dynamicSkillDirTriggers: new Set<string>(),
    discoveredSkillNames: new Set<string>(),
    toolDecisions: undefined,

    // ===== 显式 opt-in 共享 =====

    // setAppState: 默认 no-op，除非 shareSetAppState = true
    setAppState: overrides?.shareSetAppState
    ? parentContext.setAppState
    : () => {},

    // setResponseLength: 默认 no-op，除非 shareSetResponseLength = true
    setResponseLength: overrides?.shareSetResponseLength
    ? parentContext.setResponseLength
    : () => {},

    // abortController: 默认创建子控制器，除非 shareAbortController = true
    abortController: overrides?.abortController ??
    (overrides?.shareAbortController
    ? parentContext.abortController
    : createChildAbortController(parentContext.abortController)),

    // ===== 始终 no-op 的 UI 回调 =====
    addNotification: undefined,
    setToolJSX: undefined,
    setStreamMode: undefined,
    setSDKStatus: undefined,
    openMessageSelector: undefined,

    // ===== 始终共享的 =====
    // 任务注册必须到达根 store（即使 setAppState 是 no-op）
    setAppStateForTasks:
    parentContext.setAppStateForTasks ?? parentContext.setAppState,
    // Attribution 是作用域化的函数式回调，安全共享
    updateAttributionState: parentContext.updateAttributionState,
  }
}
```

** 巧思1：setAppStateForTasks 始终共享**

```typescript
// 注释原文：
// Task registration/kill must always reach the root store, even when
// setAppState is a no-op — otherwise async agents' background bash tasks
// are never registered and never killed (PPID=1 zombie).
```

即使异步 Agent 的 `setAppState` 是 no-op（不影响 UI），
它的后台 Bash 任务仍然需要注册到根 store，否则：
- 任务不会出现在 `claude ps` 中
- Agent 结束时无法 kill 这些任务
- 进程退出后变成 PPID=1 的僵尸进程

** 巧思2：contentReplacementState 克隆而非新建**

```typescript
// 注释原文：
// Clone by default (not fresh): cache-sharing forks process parent
// messages containing parent tool_use_ids. A fresh state would see
// them as unseen and make divergent replacement decisions → wire
// prefix differs → cache miss. A clone makes identical decisions →
// cache hit.
contentReplacementState:
  overrides?.contentReplacementState ??
  (parentContext.contentReplacementState
  ? cloneContentReplacementState(parentContext.contentReplacementState)
  : undefined),
```

这是为了 prompt cache 命中：Fork 子 Agent 处理父级的消息时，
如果用全新的 state，会对同一个 tool_use_id 做出不同的替换决策，
导致发送给 API 的字节不同 → 缓存未命中。

---
## 5.3 权限模式的继承与覆盖

Agent 的权限模式有一套精细的继承规则：

```typescript
const agentGetAppState = () => {
  const state = toolUseContext.getAppState()
  let toolPermissionContext = state.toolPermissionContext

  // 规则1: Agent 可以定义自己的权限模式
  // 但是！bypassPermissions、acceptEdits、auto 模式不能被覆盖
  if (
    agentPermissionMode &&
    state.toolPermissionContext.mode !== 'bypassPermissions' &&
    state.toolPermissionContext.mode !== 'acceptEdits' &&
    !(feature('TRANSCRIPT_CLASSIFIER') &&
    state.toolPermissionContext.mode === 'auto')
  ) {
    toolPermissionContext = {
      ...toolPermissionContext,
      mode: agentPermissionMode,
    }
  }

  // 规则2: 异步 Agent 不能弹权限对话框
  // 但 bubble 模式例外（冒泡到父级终端）
  const shouldAvoidPrompts =
  canShowPermissionPrompts !== undefined
  ? !canShowPermissionPrompts
  : agentPermissionMode === 'bubble'
  ? false
  : isAsync

  // 规则3: 异步但能显示 prompt 的 Agent
  // → 先等自动化检查（classifier, hooks），再弹对话框
  if (isAsync && !shouldAvoidPrompts) {
    toolPermissionContext = {
      ...toolPermissionContext,
      awaitAutomatedChecksBeforeDialog: true,
    }
  }

  // 规则4: 工具权限作用域
  // 保留 SDK 级别的 --allowedTools（cliArg）
  // 但清除父级的 session 级别权限
  if (allowedTools !== undefined) {
    toolPermissionContext = {
      ...toolPermissionContext,
      alwaysAllowRules: {
        cliArg: state.toolPermissionContext.alwaysAllowRules.cliArg,
        session: [...allowedTools],
      },
    }
  }
}
```

**权限继承矩阵**：

```
┌──────────────────────────────────────────────────────────────┐
│ 权限模式继承规则 │
├──────────────────────────────────────────────────────────────┤
│ │
│ 父级模式 \ Agent 定义 有 permissionMode 无 │
│ ────────────────────────────────────────────── │
│ bypassPermissions 父级优先 父级 │
│ acceptEdits 父级优先 父级 │
│ auto (+ classifier) 父级优先 父级 │
│ plan Agent 覆盖 ⬇ 父级 │
│ default Agent 覆盖 ⬇ 父级 │
│ │
│ 原则: 更宽松的父级模式不能被子 Agent 收紧 │
│ 原因: 用户已经选择了信任级别，子 Agent 不应该降级 │
│ │
└──────────────────────────────────────────────────────────────┘
```

---
## 5.4 工具集解析 — 多层过滤

### 5.4.1 filterToolsForAgent — 基础过滤

```typescript
export function filterToolsForAgent({
  tools, isBuiltIn, isAsync, permissionMode,
}): Tools {
  return tools.filter(tool => {
    // 规则1: MCP 工具始终允许
    if (tool.name.startsWith('mcp__')) return true

    // 规则2: plan 模式的 Agent 允许 ExitPlanMode
    if (toolMatchesName(tool, EXIT_PLAN_MODE_V2_TOOL_NAME)
    && permissionMode === 'plan') return true

    // 规则3: 所有 Agent 禁止的工具
    if (ALL_AGENT_DISALLOWED_TOOLS.has(tool.name)) return false
    // → AskUserQuestion, EnterPlanMode, ExitPlanMode, Brief

    // 规则4: 自定义 Agent 额外禁止的工具
    if (!isBuiltIn && CUSTOM_AGENT_DISALLOWED_TOOLS.has(tool.name))
    return false

    // 规则5: 异步 Agent 只允许白名单工具
    if (isAsync && !ASYNC_AGENT_ALLOWED_TOOLS.has(tool.name)) {
      // 特例: Swarm 模式的 in-process teammate
      if (isAgentSwarmsEnabled() && isInProcessTeammate()) {
        if (toolMatchesName(tool, AGENT_TOOL_NAME)) return true
        if (IN_PROCESS_TEAMMATE_ALLOWED_TOOLS.has(tool.name)) return true
      }
      return false
    }
    return true
})
}
```

### 5.4.2 resolveAgentTools — Agent 定义级过滤

```typescript
export function resolveAgentTools(
  agentDefinition, availableTools, isAsync, isMainThread,
): ResolvedAgentTools {
  // 步骤1: 基础过滤（上面的 filterToolsForAgent）
  const filteredAvailableTools = isMainThread
  ? availableTools // 主线程跳过过滤！
  : filterToolsForAgent({ ... })

  // 步骤2: 移除 disallowedTools
  const allowedAvailableTools = filteredAvailableTools.filter(
    tool => !disallowedToolSet.has(tool.name),
  )

  // 步骤3: 如果 tools 是 ['*'] 或 undefined → 允许所有
  if (hasWildcard) return { resolvedTools: allowedAvailableTools }

  // 步骤4: 精确匹配 Agent 定义的工具列表
  for (const toolSpec of agentTools) {
    const { toolName, ruleContent } = permissionRuleValueFromString(toolSpec)

    // 特殊: Agent 工具可以携带 allowedAgentTypes 元数据
    // 例如: "Agent(worker, researcher)" → 只允许这两种子 Agent
    if (toolName === AGENT_TOOL_NAME && ruleContent) {
      allowedAgentTypes = ruleContent.split(',').map(s => s.trim())
    }

    const tool = availableToolMap.get(toolName)
    if (tool) validTools.push(toolSpec)
    else invalidTools.push(toolSpec)
  }
}
```

** 巧思3：工具规格中的元数据**

```
工具规格不只是工具名，还可以携带元数据：

"Agent" → 允许 Agent 工具，所有子 Agent 类型
"Agent(worker, researcher)" → 允许 Agent 工具，但只能创建 worker 和 researcher
"Bash(npm test)" → 允许 Bash 工具，但只能运行 npm test
"Edit(src/**)" → 允许 Edit 工具，但只能编辑 src/ 下的文件

这通过 permissionRuleValueFromString() 解析：
  "Agent(worker, researcher)" → { toolName: "Agent", ruleContent: "worker, researcher" }
```

---
## 5.5 Fork 子 Agent — Prompt Cache 共享的秘密

Fork 是 Agent 系统中最精巧的模式。它的核心目标是**共享父级的 prompt cache**：

```
父级 API 请求:
  system_prompt + tools + messages[0..N] + user_message
  ├── 缓存前缀 ──────────────────────┤

Fork 子 Agent 的 API 请求:
  system_prompt + tools + messages[0..N] + fork_prompt
  ├── 相同的缓存前缀 ────────────────┤
  → 缓存命中！只付 fork_prompt 的增量 token
```

### 5.5.1 为什么 Fork 需要 useExactTools?

```typescript
// runAgent.ts 中：
const resolvedTools = useExactTools
  ? availableTools // Fork: 直接使用父级的工具列表
  : resolveAgentTools(...) // 普通: 经过过滤

const agentOptions = {
  // Fork: 继承父级的 thinkingConfig
  thinkingConfig: useExactTools
    ? toolUseContext.options.thinkingConfig
    : { type: 'disabled' }, // 普通: 禁用 thinking（省 token）

  // Fork: 继承父级的 isNonInteractiveSession
  isNonInteractiveSession: useExactTools
    ? toolUseContext.options.isNonInteractiveSession
    : isAsync ? true : (toolUseContext.options.isNonInteractiveSession ?? false),
}
```

**每一个差异都会导致缓存未命中**：
- 工具列表不同 → 缓存未命中
- thinkingConfig 不同 → 缓存未命中
- 消息前缀不同 → 缓存未命中

所以 Fork 必须精确复制父级的所有缓存关键参数。

### 5.5.2 querySource 持久化

```typescript
// Fork 需要在 options 中保存 querySource
// 因为递归 fork 守卫检查 options.querySource === 'agent:builtin:fork'
// 这个检查在 autocompact 后仍然有效（autocompact 重写消息但不重写 options）
...(useExactTools && { querySource }),
```

** 巧思4：为什么不检查消息内容？**

因为 autocompact 会重写消息！如果递归 fork 守卫只检查消息中的 fork 标记，
autocompact 后标记消失，守卫失效，可能导致无限递归 fork。

---
## 5.6 runAgent 的完整生命周期

### 5.6.1 初始化阶段（12 步）

```
1. getAgentModel() → 解析模型（Agent定义 > 父级 > 默认）
2. createAgentId() → 生成唯一 ID
3. registerPerfettoAgent() → 注册到性能追踪（层级可视化）
4. filterIncompleteToolCalls() → 过滤不完整的工具调用
5. getUserContext() / getSystemContext() → 获取上下文
6. 解析 userContext → 是否省略 CLAUDE.md
7. 解析 systemContext → 是否省略 gitStatus
8. 构建 agentGetAppState → 权限模式覆盖
9. resolveAgentTools() → 工具集解析
10. getAgentSystemPrompt() → 系统提示词
11. executeSubagentStartHooks() → 启动钩子
12. initializeAgentMcpServers() → MCP 服务器
```

### 5.6.2 filterIncompleteToolCalls — 防止 API 错误

当 Fork 子 Agent 继承父级的消息历史时，可能包含不完整的工具调用
（tool_use 没有对应的 tool_result）：

```typescript
export function filterIncompleteToolCalls(messages: Message[]): Message[] {
  // 1. 收集所有有结果的 tool_use_id
  const toolUseIdsWithResults = new Set<string>()
  for (const message of messages) {
    if (message?.type === 'user') {
      for (const block of message.message.content) {
        if (block.type === 'tool_result') {
          toolUseIdsWithResults.add(block.tool_use_id)
        }
      }
    }
  }

  // 2. 过滤掉包含无结果 tool_use 的 assistant 消息
  return messages.filter(message => {
    if (message?.type === 'assistant') {
      const hasIncompleteToolCall = message.message.content.some(
        block => block.type === 'tool_use' && !toolUseIdsWithResults.has(block.id),
      )
      return !hasIncompleteToolCall
    }
    return true
})
}
```

**为什么会有不完整的工具调用？**

当用户在工具执行过程中按 Ctrl+C，assistant 消息已经包含了 tool_use 块，
但 tool_result 还没有生成。如果把这些消息传给子 Agent，API 会返回错误。

### 5.6.3 上下文优化 — 省略不需要的信息

```typescript
// Explore/Plan Agent 省略 CLAUDE.md
// 注释原文：
// Read-only agents (Explore, Plan) don't act on commit/PR/lint rules from
// CLAUDE.md — the main agent has full context and interprets their output.
// Dropping claudeMd here saves ~5-15 Gtok/week across 34M+ Explore spawns.
const shouldOmitClaudeMd =
  agentDefinition.omitClaudeMd &&
  !override?.userContext &&
  getFeatureValue_CACHED_MAY_BE_STALE('tengu_slim_subagent_claudemd', true)

// Explore/Plan Agent 省略 gitStatus
// 注释原文：
// Explore/Plan are read-only search agents — the parent-session-start
// gitStatus (up to 40KB, explicitly labeled stale) is dead weight. If they
// need git info they run `git status` themselves and get fresh data.
// Saves ~1-3 Gtok/week fleet-wide.
```

** 巧思5：规模化的 token 节省**

```
CLAUDE.md 大小: ~2-10KB (约 500-2500 tokens)
Explore Agent 每周调用: 34M+ 次
每次节省: ~1000 tokens
每周节省: 34M × 1000 = 34 Gtok (34 billion tokens)

gitStatus 大小: 最大 40KB
每周节省: ~1-3 Gtok

总计: 每周节省 5-15 Gtok
```

这就是为什么注释中精确记录了节省量 — 这些优化在规模化时影响巨大。

---
## 5.7 MCP 服务器管理 — 共享 vs 专属

Agent 可以定义自己的 MCP 服务器，有两种方式：

```typescript
async function initializeAgentMcpServers(
  agentDefinition, parentClients,
) {
  for (const spec of agentDefinition.mcpServers) {
    if (typeof spec === 'string') {
      // 方式1: 引用已有服务器（共享连接）
      // 使用 memoized 的 connectToServer → 复用父级的连接
      // Agent 结束时不清理（因为是共享的）
      name = spec
      config = getMcpConfigByName(spec)
      isNewlyCreated = false
    } else {
      // 方式2: 内联定义（Agent 专属）
      // 创建新连接
      // Agent 结束时清理
      const [serverName, serverConfig] = Object.entries(spec)[0]
      name = serverName
      config = { ...serverConfig, scope: 'dynamic' }
      isNewlyCreated = true
    }
  }

  // 清理函数只清理新创建的连接
  const cleanup = async () => {
    for (const client of newlyCreatedClients) {
      if (client.type === 'connected') {
        await client.cleanup()
      }
    }
  }

  // 返回合并的客户端列表
  return {
    clients: [...parentClients, ...agentClients],
    tools: agentTools,
    cleanup,
  }
}
```

** 巧思6：Plugin-only 策略下的信任分级**

```typescript
// 当 MCP 被锁定为 plugin-only 时：
// - 用户创建的 Agent → 跳过 MCP 服务器
// - Plugin/built-in/policySettings Agent → 允许 MCP
const agentIsAdminTrusted = isSourceAdminTrusted(agentDefinition.source)
if (isRestrictedToPluginOnly('mcp') && !agentIsAdminTrusted) {
  return { clients: parentClients, tools: [], cleanup: async () => {} }
}
```

这确保了管理员批准的 Agent（plugin 提供的）可以使用 MCP，
而用户自定义的 Agent 不能绕过 MCP 限制。

---
## 5.8 Skill 预加载 — 三级名称解析

Agent 可以在 frontmatter 中声明需要预加载的 skills：

```typescript
function resolveSkillName(
  skillName: string,
  allSkills: Command[],
  agentDefinition: AgentDefinition,
): string | null {
  // 策略1: 精确匹配（name, userFacingName, aliases）
  if (hasCommand(skillName, allSkills)) return skillName

  // 策略2: 用 Agent 的 plugin 前缀补全
  // 例如: Agent 类型 "my-plugin:my-agent"，skill 名 "my-skill"
  // → 尝试 "my-plugin:my-skill"
  const pluginPrefix = agentDefinition.agentType.split(':')[0]
  if (pluginPrefix) {
    const qualifiedName = `${pluginPrefix}:${skillName}`
    if (hasCommand(qualifiedName, allSkills)) return qualifiedName
  }

  // 策略3: 后缀匹配
  // 找任何以 ":skillName" 结尾的命令
  const suffix = `:${skillName}`
  const match = allSkills.find(cmd => cmd.name.endsWith(suffix))
  if (match) return match.name

  return null
}
```

** 巧思7：渐进式名称解析**

```
用户写: skills: ["my-skill"]

解析过程:
  1. 精确匹配 "my-skill" → 没找到
  2. 补全为 "my-plugin:my-skill" → 找到了！
  3. (如果还没找到) 后缀匹配 ":my-skill" → 可能找到 "other-plugin:my-skill"

这让 Agent 作者可以用简短的名字引用 skill，
而不需要写完整的命名空间路径。
```

---
## 5.9 清理流程 — 为什么每一步都不能少

```typescript
try {
  for await (const message of query({ ... })) {
    // ... 执行 Agent ...
  }
} finally {
  // 1. 清理 Agent 专属 MCP 服务器
  await mcpCleanup()
  // → 不清理：MCP 进程泄漏，端口占用

  // 2. 清理 session hooks
  if (agentDefinition.hooks) {
    clearSessionHooks(rootSetAppState, agentId)
  }
  // → 不清理：Agent 的 hooks 在 Agent 结束后继续触发

  // 3. 清理 prompt cache tracking
  if (feature('PROMPT_CACHE_BREAK_DETECTION')) {
    cleanupAgentTracking(agentId)
  }
  // → 不清理：cache break 检测的内存泄漏

  // 4. 释放文件缓存
  agentToolUseContext.readFileState.clear()
  // → 不清理：克隆的文件缓存占用内存

  // 5. 释放消息数组
  initialMessages.length = 0
  // → 不清理：fork 上下文消息占用内存
  // 注意：.length = 0 比 = [] 更好，因为它释放元素引用让 GC 回收

  // 6. 释放 Perfetto 追踪
  unregisterPerfettoAgent(agentId)
  // → 不清理：性能追踪注册表泄漏

  // 7. 释放 transcript 子目录映射
  clearAgentTranscriptSubdir(agentId)
  // → 不清理：transcript 路由表泄漏

  // 8. 释放 todos 条目
  rootSetAppState(prev => {
    if (!(agentId in prev.todos)) return prev
    const { [agentId]: _removed, ...todos } = prev.todos
    return { ...prev, todos }
})
// → 不清理：每个 Agent 留下一个空的 todos 条目
// 注释原文："Whale sessions spawn hundreds of agents; each orphaned
// key is a small leak that adds up."

// 9. 终止后台 bash 任务
killShellTasksForAgent(agentId, ...)
// → 不清理：后台 shell 进程变成 PPID=1 僵尸进程

// 10. 终止 Monitor MCP 任务
if (feature('MONITOR_TOOL')) {
  mcpMod.killMonitorMcpTasksForAgent(agentId, ...)
}
// → 不清理：监控进程泄漏
}
```

** 巧思8：finally 块确保清理**

所有清理都在 `finally` 块中，确保即使 Agent 被中断（AbortError）、
抛出异常、或正常完成，清理都会执行。这是防御性编程的典范。

---
## 5.10 异步 Agent 生命周期

### 5.10.1 runAsyncAgentLifecycle — 完整的后台管理

```
spawn
 │
 ├── 创建 ProgressTracker
 ├── 可选：启动 AgentSummarization（定期生成摘要）
 │
 ▼
执行循环
 │ for await (const message of makeStream()) {
 │ ├── 追加到 agentMessages
 │ ├── 如果 UI 持有任务 → 实时追加到 AppState
 │ ├── updateProgressFromMessage() → 更新进度
 │ └── emitTaskProgress() → SDK 进度事件
 │ }
 │
 ├── 成功路径:
 │ ├── stopSummarization()
 │ ├── finalizeAgentTool() → 提取结果
 │ ├── completeAsyncAgent() → 标记完成 (先！)
 │ ├── classifyHandoffIfNeeded() → 安全检查 (后)
 │ ├── getWorktreeResult() → worktree 清理 (后)
 │ └── enqueueAgentNotification() → 通知
 │
 ├── AbortError 路径 (用户 kill):
 │ ├── killAsyncAgent() → 标记为 killed
 │ ├── extractPartialResult() → 提取部分结果
 │ └── enqueueAgentNotification(status: 'killed')
 │
 └── 其他错误路径:
 ├── failAsyncAgent() → 标记失败
 └── enqueueAgentNotification(status: 'failed')
```

### 5.10.2 Handoff 安全分类

在 auto 模式下，子 Agent 完成后需要安全审查：

```typescript
export async function classifyHandoffIfNeeded({
  agentMessages, tools, toolPermissionContext, abortSignal, subagentType,
}) {
  if (toolPermissionContext.mode !== 'auto') return null

  // 构建分类器 transcript
  const agentTranscript = buildTranscriptForClassifier(agentMessages, tools)

  // 调用 YOLO 分类器
  const classifierResult = await classifyYoloAction(
    agentMessages,
    {
      role: 'user',
      content: [{
        type: 'text',
        text: "Sub-agent has finished and is handing back control to the main agent. "
        + "Review the sub-agent's work based on the block rules and let the main "
        + "agent know if any file is dangerous.",
    }],
  },
  tools,
  toolPermissionContext,
  abortSignal,
)

if (classifierResult.shouldBlock) {
  if (classifierResult.unavailable) {
    // 分类器不可用 → 允许但附带警告
    return `Note: The safety classifier was unavailable. Please carefully verify.`
  }
  // 分类器标记为危险 → 附带安全警告
  return `SECURITY WARNING: This sub-agent performed actions that may violate "
    + "security policy. Reason: ${classifierResult.reason}.`
}
return null
}
```

**设计要点**：
- 只在 auto 模式下运行（其他模式用户已经明确选择了信任级别）
- 分类器不可用时不阻塞，而是附带警告
- 警告注入到结果中，让主 Agent 可以看到并决定如何处理

---
## 5.11 Transcript 记录 — 增量追加

Agent 的执行过程被记录到 sidechain transcript 中：

```typescript
// 记录初始消息
void recordSidechainTranscript(initialMessages, agentId)

// 记录每条新消息（增量追加，O(1)）
let lastRecordedUuid: UUID | null = initialMessages.at(-1)?.uuid ?? null

for await (const message of query({ ... })) {
  if (isRecordableMessage(message)) {
    // 只记录新消息，附带 parent UUID 维护链式关系
    await recordSidechainTranscript(
      [message],
      agentId,
      lastRecordedUuid, // parent chain
    )
    if (message.type !== 'progress') {
      lastRecordedUuid = message.uuid
    }
  }
}
```

** 巧思10：progress 消息不更新 lastRecordedUuid**

Progress 消息是临时的（如 Bash 的实时输出），不应该成为链的一部分。
如果 progress 更新了 lastRecordedUuid，后续的 assistant 消息会以 progress 为 parent，
导致 resume 时链式关系错误。

### 5.11.1 API Metrics 转发

```typescript
// 子 Agent 的 API 请求指标转发给父级
if (
  message.type === 'stream_event' &&
  message.event.type === 'message_start' &&
  message.ttftMs != null
) {
  toolUseContext.pushApiMetricsEntry?.(message.ttftMs)
  continue // 不 yield 给调用者
}
```

这让父级的 TTFT/OTPS 显示在子 Agent 执行期间也能更新，
用户可以看到实时的性能指标。

---
## 5.12 Cache Eviction Hint — 主动释放缓存

```typescript
// finalizeAgentTool 中：
const lastRequestId = lastAssistantMessage.requestId
if (lastRequestId) {
  logEvent('tengu_cache_eviction_hint', {
    scope: 'subagent_end',
    last_request_id: lastRequestId,
})
}
```

** 巧思11：主动告诉推理服务器释放缓存**

子 Agent 结束后，它的 prompt cache chain 不再需要了。
通过发送 eviction hint，推理服务器可以提前释放这些缓存，
为其他请求腾出空间。

这在大规模部署中很重要 — 每个子 Agent 可能占用 5-50MB 的缓存空间，
一个长会话可能产生数十个子 Agent。

---
## 5.13 localDenialTracking — 异步 Agent 的权限拒绝计数

```typescript
// createSubagentContext 中：
localDenialTracking: overrides?.shareSetAppState
  ? parentContext.localDenialTracking
  : createDenialTrackingState(),
```

**为什么异步 Agent 需要本地的 denial tracking？**

异步 Agent 的 `setAppState` 是 no-op，所以全局的 denial counter 不会更新。
如果没有本地 tracking，Agent 可以无限次重试被拒绝的操作。

本地 tracking 确保即使 setAppState 是 no-op，denial counter 仍然在本地累积，
达到阈值后 Agent 会停止重试。

---
## 5.14 Coordinator 模式 — 编排者与执行者的分离

Coordinator 模式是多 Agent 系统的高级形态。它将 Claude Code 从"一个 Agent 做所有事"
变成"一个编排者 + 多个 Worker"的架构。

### 5.14.1 模式切换

```typescript
// coordinatorMode.ts
export function isCoordinatorMode(): boolean {
  if (feature('COORDINATOR_MODE')) {
    return isEnvTruthy(process.env.CLAUDE_CODE_COORDINATOR_MODE)
  }
  return false
}
```

通过环境变量 `CLAUDE_CODE_COORDINATOR_MODE=1` 启用。注意它是**运行时读取**的
（没有缓存），这让 `matchSessionMode()` 可以在恢复会话时动态切换模式。

### 5.14.2 会话模式匹配

```typescript
// 恢复会话时，如果当前模式和会话存储的模式不一致，自动切换
export function matchSessionMode(
  sessionMode: 'coordinator' | 'normal' | undefined,
): string | undefined {
  const currentIsCoordinator = isCoordinatorMode()
  const sessionIsCoordinator = sessionMode === 'coordinator'

  if (currentIsCoordinator === sessionIsCoordinator) return undefined

  // 直接翻转环境变量
  if (sessionIsCoordinator) {
    process.env.CLAUDE_CODE_COORDINATOR_MODE = '1'
  } else {
    delete process.env.CLAUDE_CODE_COORDINATOR_MODE
  }
  return sessionIsCoordinator
    ? 'Entered coordinator mode to match resumed session.'
    : 'Exited coordinator mode to match resumed session.'
}
```

这解决了一个实际问题：用户在 coordinator 模式下创建了会话，关闭后以普通模式恢复，
Worker 的 task-notification 会无法被正确处理。

### 5.14.3 Coordinator 的 System Prompt 设计

Coordinator 的 system prompt 长达 ~250 行，定义了完整的工作流程：

```
Coordinator 的角色定义：
  1. 你是编排者，不是执行者
  2. 直接回答能回答的问题，不要委派简单工作
  3. Worker 结果以 <task-notification> XML 到达
  4. 永远不要伪造或预测 Agent 结果

可用工具：
  - Agent (spawn worker)
  - SendMessage (continue worker)
  - TaskStop (stop worker)
  - subscribe/unsubscribe_pr_activity

任务工作流（4 阶段）：
  1. Research → Workers 并行调查
  2. Synthesis → Coordinator 理解并综合
  3. Implementation → Workers 执行
  4. Verification → Workers 验证
```

**巧思12：Synthesis 阶段是 Coordinator 最重要的职责**

```
// System prompt 中的反模式示例：
// 坏: "Based on your findings, fix the auth bug"  ← 懒惰委派
// 好: "Fix the null pointer in src/auth/validate.ts:42.
//      The user field on Session is undefined when sessions expire
//      but the token remains cached. Add a null check before user.id
//      access — if null, return 401 with 'Session expired'."
```

Coordinator 必须**理解** Worker 的研究结果，然后写出具体的实施规格。
这不是简单的转发，而是真正的综合分析。

### 5.14.4 Worker 上下文注入

```typescript
export function getCoordinatorUserContext(
  mcpClients: ReadonlyArray<{ name: string }>,
  scratchpadDir?: string,
): { [k: string]: string } {
  if (!isCoordinatorMode()) return {}

  // 告诉 Coordinator 它的 Worker 有哪些工具
  const workerTools = Array.from(ASYNC_AGENT_ALLOWED_TOOLS)
    .filter(name => !INTERNAL_WORKER_TOOLS.has(name))
    .sort()
    .join(', ')

  let content = `Workers spawned via the ${AGENT_TOOL_NAME} tool have access to these tools: ${workerTools}`

  // MCP 服务器信息
  if (mcpClients.length > 0) {
    const serverNames = mcpClients.map(c => c.name).join(', ')
    content += `\n\nWorkers also have access to MCP tools from connected MCP servers: ${serverNames}`
  }

  // Scratchpad 目录（跨 Worker 共享知识）
  if (scratchpadDir && isScratchpadGateEnabled()) {
    content += `\n\nScratchpad directory: ${scratchpadDir}
Workers can read and write here without permission prompts.
Use this for durable cross-worker knowledge.`
  }

  return { workerToolsContext: content }
}
```

**巧思13：Scratchpad 目录**

Scratchpad 是 Coordinator 模式的关键基础设施：
- Worker 之间不能直接通信
- 但它们可以通过 Scratchpad 目录共享文件
- 无需权限提示（已预授权）
- Coordinator 可以指导 Worker 在这里存放中间结果

### 5.14.5 Continue vs Spawn 决策

Coordinator 的 system prompt 中有一个精心设计的决策矩阵：

```
| 场景                           | 机制          | 原因                           |
|-------------------------------|--------------|-------------------------------|
| 研究的文件正好是要编辑的        | Continue     | Worker 已有文件上下文           |
| 研究范围广但实施范围窄          | Spawn fresh  | 避免拖入探索噪音               |
| 纠正失败或扩展近期工作          | Continue     | Worker 有错误上下文             |
| 验证另一个 Worker 写的代码      | Spawn fresh  | 验证者应该用全新视角            |
| 第一次实施用了错误方法          | Spawn fresh  | 错误方法的上下文会锚定重试      |
| 完全无关的任务                  | Spawn fresh  | 没有可复用的上下文              |
```

核心原则：**上下文重叠度高 → Continue，低 → Spawn fresh**。

### 5.14.6 Worker 通信机制详解

Worker 之间不能直接通信，所有通信都通过 Coordinator 中转。理解这三种通信机制的区别至关重要：

```
┌─────────────────────────────────────────────────────────────────────────┐
│ 三种 Worker 通信机制对比                                               │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│ 1. task-notification (异步消息)                                        │
│    方向: Worker → Coordinator                                          │
│    触发: Worker 完成/失败/被 kill                                     │
│    格式: <task-notification>                                             │
│          <task_id>worker-123</task_id>                                   │
│          <status>completed</status>                                      │
│          <result>修复了 3 个文件...</result>                              │
│          </task-notification>                                            │
│    时序: 通过消息队列异步到达，Coordinator 在下次迭代时消费           │
│    丢失处理: 不会丢失，队列持久化在内存中                             │
│                                                                         │
│ 2. SendMessage (主动消息)                                               │
│    方向: Coordinator → Worker                                          │
│    触发: Coordinator 主动发送指令或追加上下文                       │
│    用途: 继续已有 Worker 的工作（而不是 spawn 新的）                   │
│    时序: 同步注入到 Worker 的消息队列，Worker 在下次迭代时看到        │
│    限制: 只能发给还活着的 Worker                                     │
│                                                                         │
│ 3. Scratchpad 文件 (共享存储)                                        │
│    方向: Worker ↔ Worker (通过文件系统)                              │
│    触发: Worker 写文件，另一个 Worker 读文件                          │
│    用途: 共享研究结果、中间数据、配置文件                            │
│    时序: 无保证！Worker A 写入后，Worker B 不一定立即看到            │
│    限制: 无锁机制，需要 Coordinator 协调读写顺序                    │
│                                                                         │
│ 实践原则:                                                              │
│ - 控制流用 task-notification + SendMessage                              │
│ - 数据流用 Scratchpad                                                   │
│ - Coordinator 负责协调时序，不依赖 Worker 自己同步                  │
└─────────────────────────────────────────────────────────────────────────┘
```

### 5.14.7 完整编排案例：修复跨 3 个文件的认证 Bug

以下是一个端到端的 Coordinator 编排追踪，展示实际的任务分解、分配、合并过程：

```
用户输入: "登录后 session 过期但 token 还在缓存中，导致空指针崩溃"

═══ 阶段 1: Research (并行研究) ═══

Coordinator 思考:
  "这涉及 session 管理、token 缓存、和认证验证三个领域。
   我需要先理解代码结构，再制定修复方案。"

T+0s     Coordinator 调用 Agent("研究 session 生命周期管理的代码结构")
         → spawn Worker-A (async)

T+0.1s   Coordinator 调用 Agent("研究 token 缓存机制和过期处理逻辑")
         → spawn Worker-B (async)

T+0.1s   Coordinator 调用 Agent("研究认证验证中对 user 字段的空值处理")
         → spawn Worker-C (async)

         三个 Worker 并行执行，Coordinator 等待...

T+15s    <task-notification> Worker-A 完成
         结果: "Session 定义在 src/auth/session.ts，
                expire() 方法清除 session 但不清除 tokenCache，
                Session 类型的 user 字段在过期后变为 undefined"

T+20s    <task-notification> Worker-B 完成
         结果: "tokenCache 在 src/auth/tokenCache.ts，
                使用 sessionId 作为 key，
                没有监听 session 过期事件，
                过期的 session 的 token 会一直留在缓存中"

T+22s    <task-notification> Worker-C 完成
         结果: "src/auth/validate.ts:42 行直接访问 user.id，
                没有 null check，
                当 session 过期但 token 还在缓存时，
                validate() 用缓存的 token 找到过期的 session，
                session.user 是 undefined → 崩溃"

═══ 阶段 2: Synthesis (综合分析) ═══

Coordinator 思考 (这是最关键的阶段):
  "根据三个 Worker 的研究，我现在理解了完整的 bug 链路：
   1. session.expire() 清除 session 但不通知 tokenCache
   2. tokenCache 保留了过期 session 的 token
   3. validate() 用缓存 token 找到过期 session，访问 user.id 崩溃

   修复方案：
   - src/auth/session.ts: expire() 时发出事件
   - src/auth/tokenCache.ts: 监听过期事件，清除对应 token
   - src/auth/validate.ts: 添加 null check 作为防御性编程"

═══ 阶段 3: Implementation (并行实施) ═══

决策: Worker-C 研究的文件正好是要编辑的 → Continue Worker-C
       其他两个文件的修复范围窄 → Spawn fresh

T+25s    Coordinator 调用 SendMessage(Worker-C,
           "在 src/auth/validate.ts:42 添加 null check：
            if (!session?.user) return { status: 401, error: 'Session expired' }")
         → Continue Worker-C

T+25.1s  Coordinator 调用 Agent(
           "在 src/auth/session.ts 的 expire() 方法中，
            添加 this.emit('expired', this.id)，
            并在 src/auth/tokenCache.ts 中监听该事件，
            收到时调用 cache.delete(sessionId)")
         → spawn Worker-D (async)

T+35s    <task-notification> Worker-C 完成 (添加了 null check)
T+40s    <task-notification> Worker-D 完成 (添加了事件机制)

═══ 阶段 4: Verification (独立验证) ═══

决策: 验证者应该用全新视角 → Spawn fresh

T+42s    Coordinator 调用 Agent(
           "验证以下修复是否完整：
            1. 运行现有测试 npm test
            2. 检查 session.expire() 是否正确发出事件
            3. 检查 tokenCache 是否正确监听并清除
            4. 检查 validate.ts 的 null check 是否覆盖所有路径")
         → spawn Worker-E (async, 全新视角)

T+60s    <task-notification> Worker-E 完成
         结果: "所有测试通过，修复完整"

Coordinator 回复用户: "已修复。根本原因是 session 过期时不清除 token 缓存..."

总计: 5 个 Worker，~60 秒，串行执行需要 ~3 分钟
```

**案例中的关键决策点**：

| 决策 | 选择 | 原因 |
|------|------|------|
| 研究阶段用几个 Worker？ | 3 个 | 每个关注一个领域，避免上下文污染 |
| Worker-C Continue vs Spawn？ | Continue | 它已经读过 validate.ts，上下文重叠度高 |
| Worker-D 为什么不继续 Worker-A？ | Spawn fresh | 研究范围广但实施范围窄，避免探索噪音 |
| 验证为什么用新 Worker？ | Spawn fresh | 验证者应该用全新视角，不被实施者的思路影响 |
| Coordinator 自己做了什么？ | Synthesis | 理解 bug 链路，制定具体修复规格，不是懒惰转发 |

---
## 本章小结：工程巧思清单

| # | 巧思 | 解决的问题 |
|---|------|------------|
| 1 | setAppStateForTasks 始终共享 | 防止后台任务变成僵尸进程 |
| 2 | contentReplacementState 克隆 | 保证 Fork 的 prompt cache 命中 |
| 3 | 工具规格中的元数据 | 精细控制子 Agent 的工具权限 |
| 4 | querySource 持久化 | 防止 autocompact 后递归 fork 守卫失效 |
| 5 | 规模化 token 节省 | 省略 CLAUDE.md 每周节省 5-15 Gtok |
| 6 | Plugin-only 信任分级 | 管理员 Agent 可用 MCP，用户 Agent 不行 |
| 7 | 三级名称解析 | 让 Agent 作者用简短名字引用 skill |
| 8 | finally 块清理 | 确保所有退出路径都清理资源 |
| 9 | complete 在 classify 之前 | 防止慢操作导致死锁 |
| 10 | progress 不更新 parent chain | 防止 resume 时链式关系错误 |
| 11 | Cache eviction hint | 主动释放子 Agent 的缓存空间 |
| 12 | Synthesis 阶段 | Coordinator 必须理解再委派，不能懒惰转发 |
| 13 | Scratchpad 目录 | 跨 Worker 共享知识，无需权限提示 |

---
## 5.15 可迁移的设计模式 — 带走这些，用在你自己的 Agent 系统中

### 模式1：Capability-based Security（基于能力的安全模型）

**在 Claude Code 中**：`createSubagentContext` — 默认隔离所有状态，调用者必须显式 opt-in 共享。

**通用形式**：
```
创建子组件时：
  默认: 所有能力 = 无 (或 no-op)
  显式授权: 只给子组件它需要的最小能力集
```

**适用场景**：
- 浏览器沙箱（iframe 默认无权限，通过 `allow="camera"` 显式授权）
- Docker 容器（默认无网络/文件系统访问，通过 `--cap-add` 授权）
- 微服务的 API Gateway（默认拒绝所有请求，通过白名单放行）

**关键约束**：某些能力必须始终共享（如 `setAppStateForTasks`），
否则子组件的副作用无法被父级跟踪，导致资源泄漏。

### 模式2：缓存友好的 Fork（Cache-Friendly Forking）

**在 Claude Code 中**：Fork 子 Agent 精确复制父级的所有缓存关键参数，确保 prompt cache 命中。

**通用形式**：
```
创建子进程/子任务时：
  确保子任务的输入前缀与父任务完全相同
  → 共享缓存，只为增量部分付费
```

**适用场景**：
- Linux 的 fork() + CoW（子进程共享父进程的内存页，只在写入时复制）
- CDN 的分层缓存（边缘节点共享源站缓存前缀）
- Git 的 packfile（新分支共享基础对象，只存储增量）

**关键约束**：所有影响缓存 key 的参数都必须精确复制，
包括不明显的参数（如 `contentReplacementState`）。一个字节的差异就会导致缓存未命中。

### 模式3：确定性清理（Deterministic Cleanup）

**在 Claude Code 中**：`finally` 块中的 12 步清理流程，确保所有退出路径都清理资源。

**通用形式**：
```
try {
  resource = acquire()
  // ... 使用资源 ...
} finally {
  // 每个资源都有对应的清理，且清理顺序与获取顺序相反
  cleanup_N()  // 最后获取的最先清理
  cleanup_2()
  cleanup_1()
}
```

**适用场景**：
- 数据库连接池（连接必须归还，否则池耗尽）
- 文件句柄（必须关闭，否则 fd 泄漏）
- Kubernetes Pod（必须清理 sidecar，否则 Pod 卡在 Terminating）

**关键约束**：清理必须是**幂等**的（多次执行结果相同），
因为在异常场景下可能被重复调用。

### 模式4：星型拓扑编排（Star Topology Orchestration）

**在 Claude Code 中**：Coordinator 模式 — 一个编排者 + 多个 Worker，Worker 之间不直接通信。

**通用形式**：
```
         Coordinator
        /    |    \
Worker-A  Worker-B  Worker-C
(不直接通信，通过 Coordinator 中转)
```

**适用场景**：
- MapReduce（Master 分发任务，Worker 独立执行，Master 合并结果）
- 微服务的 Saga 模式（Orchestrator 协调多个服务的事务）
- CI/CD 的 Pipeline（Controller 调度多个 Stage，Stage 之间通过 artifact 共享数据）

**关键约束**：Coordinator 必须做 **Synthesis**（综合分析），不能只做转发。
如果 Coordinator 只是把用户请求拆分后直接转发给 Worker，
那它就是一个没有价值的中间层。真正的价值在于：
- 研究后综合分析，制定具体的实施规格
- 根据上下文重叠度决定 Continue vs Spawn
- 验证时用全新视角，避免确认偏差

### 模式5：权限的单调递减（Monotonic Permission Narrowing）

**在 Claude Code 中**：子 Agent 可以收窄权限，但不能放宽父级的权限。

**通用形式**：
```
父级权限 = {读, 写, 执行}
子级权限 ⊆ 父级权限  (只能是子集，不能超集)
```

**适用场景**：
- Unix 的 setuid/setgid（子进程不能获得父进程没有的权限）
- OAuth 的 scope 继承（刷新 token 的 scope 不能超过原始 token）
- AWS IAM 的 permission boundary（子角色的权限不能超过 boundary）

**关键约束**：更宽松的父级模式（如 `bypassPermissions`）不能被子级收紧，
因为用户已经明确选择了信任级别。这是一个微妙但重要的设计决策。

---

### 下一章预告
第6章将深入 **MCP 协议与扩展系统**——理解 Claude Code 如何通过 MCP 和 Skills 扩展能力。

