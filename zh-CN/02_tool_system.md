# 第2章：工具系统 — Agent 的"手和脚"

## 学习目标
1. 理解 Tool 接口的完整契约（每个方法的作用）
2. 从最简单的 GlobTool 开始，掌握工具的标准结构
3. 理解工具注册、过滤、组装的完整流程
4. 理解工具执行的生命周期

---

## 2.1 Tool 接口全景 (Tool.ts)

`Tool.ts` (28KB, 793行) 定义了所有工具必须遵循的接口。这是整个工具系统的"宪法"。

### 核心接口结构

```typescript
export type Tool<Input, Output, Progress> = {
  // ========== 身份标识 ==========
  readonly name: string; // 工具名称，如 'Glob', 'Bash', 'Read'
  aliases?: string[]; // 别名（重命名后向后兼容）
  searchHint?: string; // ToolSearch 关键词匹配提示

  // ========== Schema 定义 ==========
  readonly inputSchema: Input; // Zod schema，定义输入参数
  outputSchema?: z.ZodType; // 输出 schema
  readonly inputJSONSchema?: object; // MCP 工具的 JSON Schema

  // ========== 核心执行 ==========
  call(args, context, canUseTool, parentMessage, onProgress?)
  : Promise<ToolResult<Output>>; // 执行工具的核心方法

  // ========== 权限与安全 ==========
  validateInput?(input, context): Promise<ValidationResult>;
  checkPermissions(input, context): Promise<PermissionResult>;
  isReadOnly(input): boolean; // 是否只读操作
  isDestructive?(input): boolean; // 是否不可逆操作
  isEnabled(): boolean; // 是否启用
  isConcurrencySafe(input): boolean; // 是否可并发执行

  // ========== Prompt 生成 ==========
  description(input, options): Promise<string>; // 给 LLM 的描述
  prompt(options): Promise<string>; // 完整的工具提示词

  // ========== UI 渲染 ==========
  renderToolUseMessage(input, options): React.ReactNode; // 渲染工具调用
  renderToolResultMessage?(output, progress, options): React.ReactNode; // 渲染结果
  renderToolUseProgressMessage?(progress, options): React.ReactNode; // 渲染进度
  renderToolUseRejectedMessage?(input, options): React.ReactNode; // 渲染拒绝
  renderToolUseErrorMessage?(result, options): React.ReactNode; // 渲染错误

  // ========== 序列化 ==========
  mapToolResultToToolResultBlockParam(output, toolUseID): ToolResultBlockParam;
  toAutoClassifierInput(input): unknown; // 安全分类器输入

  // ========== 辅助方法 ==========
  userFacingName(input): string; // 用户可见名称
  getPath?(input): string; // 获取操作路径
  getToolUseSummary?(input): string; // 简短摘要
  getActivityDescription?(input): string; // 活动描述（spinner）
  maxResultSizeChars: number; // 结果最大字符数
};
```

## 2.2 buildTool() — 工具工厂函数

所有工具都通过 `buildTool()` 创建，它提供了安全的默认值：

```typescript
// Tool.ts 中的默认值
const TOOL_DEFAULTS = {
  isEnabled: () => true, // 默认启用
  isConcurrencySafe: () => false, // 默认不安全（保守）
  isReadOnly: () => false, // 默认假设会写入
  isDestructive: () => false, // 默认非破坏性
  checkPermissions: (input) => // 默认允许（交给通用权限系统）
  Promise.resolve({ behavior: 'allow', updatedInput: input }),
  toAutoClassifierInput: () => '', // 默认跳过分类器
  userFacingName: () => '', // 默认空
};

export function buildTool(def) {
  return {
    ...TOOL_DEFAULTS,
    userFacingName: () => def.name, // 默认用工具名
    ...def, // 用户定义覆盖默认值
  };
}
```

**设计哲学：Fail-Closed（安全优先）**
- `isConcurrencySafe` 默认 `false` → 不确定就不并发
- `isReadOnly` 默认 `false` → 不确定就假设会写入
- 这确保了新工具即使忘记设置这些属性，也不会造成安全问题

## 2.3 最简单的工具：GlobTool 完整解析

GlobTool 是最简单的内置工具之一，只有 ~200 行代码。让我们逐行理解：

### 文件结构
```
src/tools/GlobTool/
├── GlobTool.ts # 核心逻辑 (199行)
├── prompt.ts # 工具描述 (8行)
└── UI.tsx # 终端渲染组件
```

### prompt.ts — 给 LLM 的工具描述
```typescript
export const GLOB_TOOL_NAME = 'Glob';

export const DESCRIPTION = `
- Fast file pattern matching tool that works with any codebase size
- Supports glob patterns like "**/*.js" or "src/**/*.ts"
- Returns matching file paths sorted by modification time
- Use this tool when you need to find files by name patterns
- When you are doing an open ended search that may require multiple 
 rounds of globbing and grepping, use the Agent tool instead
`;
```

**注意最后一行！** 它引导 LLM 在复杂搜索时使用 AgentTool 而非 GlobTool。
这是 **prompt engineering 在工具描述中的应用**。

### GlobTool.ts — 核心实现逐段解析

#### 第1部分：输入/输出 Schema

```typescript
// 使用 Zod 定义输入参数
const inputSchema = lazySchema(() =>
z.strictObject({
  pattern: z.string()
  .describe('The glob pattern to match files against'),
  path: z.string().optional()
  .describe('The directory to search in. If not specified, the current '
  + 'working directory will be used. IMPORTANT: Omit this field to use '
  + 'the default directory. DO NOT enter "undefined" or "null"'),
}),
);

// 输出 Schema
const outputSchema = lazySchema(() =>
z.object({
  durationMs: z.number(), // 执行耗时
  numFiles: z.number(), // 找到的文件数
  filenames: z.array(z.string()), // 文件路径列表
  truncated: z.boolean(), // 是否被截断
}),
);
```

**关键细节：**
- `lazySchema()` 延迟初始化，避免模块加载时的循环依赖
- `z.strictObject()` 不允许额外字段（比 `z.object()` 更严格）
- `.describe()` 的文本会被发送给 LLM，是 prompt 的一部分！

#### 第2部分：工具属性声明

```typescript
export const GlobTool = buildTool({
  name: GLOB_TOOL_NAME, // 'Glob'
  searchHint: 'find files by name pattern or wildcard',
  maxResultSizeChars: 100_000, // 结果最大 100K 字符

  // 属性方法
  isConcurrencySafe() { return true; }, // 可以并发执行
  isReadOnly() { return true; }, // 只读操作

  isSearchOrReadCommand() {
    return { isSearch: true, isRead: false }; // 搜索类操作（UI 折叠）
  },

  // 获取操作路径
  getPath({ path }) {
    return path ? expandPath(path) : getCwd();
  },

  // 安全分类器输入
  toAutoClassifierInput(input) {
    return input.pattern; // 只传递 pattern，不传路径
  },
});
```

**为什么 `isConcurrencySafe` 返回 `true`？**
因为 Glob 只是读取文件系统，不修改任何状态，多个 Glob 可以安全并行执行。

#### 第3部分：输入验证

```typescript
async validateInput({ path }): Promise<ValidationResult> {
  if (path) {
    const absolutePath = expandPath(path);

    // 安全检查：阻止 UNC 路径（防止 NTLM 凭证泄露）
    if (absolutePath.startsWith('\\\\') || absolutePath.startsWith('//')) {
      return { result: true }; // 静默跳过，不报错
    }

    // 检查路径是否存在
    let stats;
    try {
      stats = await fs.stat(absolutePath);
    } catch (e) {
      if (isENOENT(e)) {
        // 路径不存在，尝试建议正确路径
        const suggestion = await suggestPathUnderCwd(absolutePath);
        return {
          result: false,
          message: `Directory does not exist: ${path}. Did you mean ${suggestion}?`,
          errorCode: 1,
        };
      }
      throw e;
    }

    // 检查是否是目录
    if (!stats.isDirectory()) {
      return { result: false, message: `Path is not a directory: ${path}`, errorCode: 2 };
    }
  }
  return { result: true };
},
```

**验证流程：**
1. UNC 路径安全检查（Windows 特有的安全问题）
2. 路径存在性检查
3. 路径类型检查（必须是目录）
4. 智能建议（"Did you mean...?"）

#### 第4部分：权限检查

```typescript
async checkPermissions(input, context): Promise<PermissionDecision> {
  const appState = context.getAppState();
  return checkReadPermissionForTool(
    GlobTool,
    input,
    appState.toolPermissionContext,
  );
},
```

GlobTool 是只读工具，所以使用通用的 `checkReadPermissionForTool()`。
这个函数会检查：
- 路径是否在允许的工作目录内
- 是否有 deny/ask/allow 规则匹配
- UNC 路径防护

#### 第5部分：核心执行逻辑

```typescript
async call(input, { abortController, getAppState, globLimits }) {
  const start = Date.now();
  const appState = getAppState();
  const limit = globLimits?.maxResults ?? 100; // 默认最多100个结果

  // 执行 glob 搜索
  const { files, truncated } = await glob(
    input.pattern,
    GlobTool.getPath(input),
    { limit, offset: 0 },
    abortController.signal, // 支持中断
    appState.toolPermissionContext, // 权限上下文
  );

  // 相对路径化（节省 token）
  const filenames = files.map(toRelativePath);

  return {
    data: {
      filenames,
      durationMs: Date.now() - start,
      numFiles: filenames.length,
      truncated,
    },
  };
},
```

**关键设计：**
- `abortController.signal` — 用户可以随时中断
- `toRelativePath` — 将绝对路径转为相对路径，节省 token 消耗
- 结果限制为 100 个文件，防止 token 爆炸

#### 第6部分：结果序列化

```typescript
mapToolResultToToolResultBlockParam(output, toolUseID) {
  // 没有结果
  if (output.filenames.length === 0) {
    return {
      tool_use_id: toolUseID,
      type: 'tool_result',
      content: 'No files found',
    };
  }
  // 有结果：每行一个文件名
  return {
    tool_use_id: toolUseID,
    type: 'tool_result',
    content: [
      ...output.filenames,
      ...(output.truncated
      ? ['(Results are truncated. Consider using a more specific path or pattern.)']
      : []),
    ].join('\n'),
  };
},
```

这个方法将工具输出转换为 Anthropic API 的 `tool_result` 格式。
注意截断提示——它引导 LLM 使用更精确的搜索条件。

## 2.4 工具注册表 (tools.ts)

`tools.ts` (390行) 是所有工具的注册中心。它决定了哪些工具对 LLM 可见。

### getAllBaseTools() — 获取所有基础工具

```typescript
export function getAllBaseTools(): Tools {
  return [
    // 核心工具（始终可用）
    AgentTool, // 子 Agent
    TaskOutputTool, // 任务输出
    BashTool, // Shell 命令
    FileReadTool, // 读文件
    FileEditTool, // 编辑文件
    FileWriteTool, // 写文件
    NotebookEditTool, // Jupyter 编辑
    WebFetchTool, // 获取网页
    TodoWriteTool, // Todo 管理
    WebSearchTool, // 网页搜索
    TaskStopTool, // 停止任务
    AskUserQuestionTool,// 询问用户
    SkillTool, // 技能调用
    EnterPlanModeTool, // 进入计划模式
    ExitPlanModeV2Tool, // 退出计划模式
    BriefTool, // 简要模式

    // 条件工具（根据环境/feature flag）
    ...(hasEmbeddedSearchTools() ? [] : [GlobTool, GrepTool]),
    ...(isAgentSwarmsEnabled() ? [TeamCreateTool, TeamDeleteTool] : []),
    ...(isWorktreeModeEnabled() ? [EnterWorktreeTool, ExitWorktreeTool] : []),
    ...(isToolSearchEnabledOptimistic() ? [ToolSearchTool] : []),

    // Feature flag 控制的实验性工具
    ...(WebBrowserTool ? [WebBrowserTool] : []),
    ...(SleepTool ? [SleepTool] : []),
    ...cronTools,

    // MCP 资源工具
    ListMcpResourcesTool,
    ReadMcpResourceTool,
  ];
}
```

**注意 `hasEmbeddedSearchTools()` 的逻辑：**
Anthropic 内部版本（ant build）将 ripgrep 等工具嵌入到 Bun 二进制中，
此时 Glob/Grep 工具就不需要了，因为 BashTool 中的 find/grep 已经被别名到快速工具。

### getTools() — 获取过滤后的工具

```typescript
export const getTools = (permissionContext): Tools => {
  // 简单模式：只有 Bash + Read + Edit
  if (isEnvTruthy(process.env.CLAUDE_CODE_SIMPLE)) {
    return filterToolsByDenyRules([BashTool, FileReadTool, FileEditTool], permissionContext);
  }

  // 正常模式
  const tools = getAllBaseTools();

  // 1. 过滤被 deny 规则禁止的工具
  let allowedTools = filterToolsByDenyRules(tools, permissionContext);

  // 2. REPL 模式下隐藏原始工具（它们在 VM 内可用）
  if (isReplModeEnabled()) {
    allowedTools = allowedTools.filter(tool => !REPL_ONLY_TOOLS.has(tool.name));
  }

  // 3. 过滤未启用的工具
  return allowedTools.filter(tool => tool.isEnabled());
};
```

### assembleToolPool() — 合并内置工具和 MCP 工具

```typescript
export function assembleToolPool(permissionContext, mcpTools): Tools {
  const builtInTools = getTools(permissionContext);
  const allowedMcpTools = filterToolsByDenyRules(mcpTools, permissionContext);

  // 内置工具排在前面（prompt cache 稳定性）
  // 同名时内置工具优先（uniqBy 保留第一个）
  return uniqBy(
    [...builtInTools].sort(byName)
    .concat(allowedMcpTools.sort(byName)),
    'name',
  );
}
```

**为什么要排序？**
Anthropic API 有 prompt cache 机制。工具列表的顺序变化会导致 cache miss。
排序确保了顺序稳定，最大化 cache 命中率。

## 2.5 工具执行生命周期

当 Claude 返回一个 `tool_use` 块时，执行流程如下：

```
Claude API 返回: { type: 'tool_use', name: 'Glob', input: { pattern: '**/*.ts' } }
 │
 ▼
1. 查找工具: findToolByName(tools, 'Glob') → GlobTool
 │
 ▼
2. 输入解析: tool.inputSchema.parse(input)
 │ └── Zod 验证输入格式
 │
 ▼
3. 输入验证: tool.validateInput(parsedInput, context)
 │ └── 检查路径是否存在、是否是目录等
 │
 ▼
4. 权限检查: hasPermissionsToUseTool(tool, input, context)
 │ ├── 4a. 检查 deny rules → 被禁止？
 │ ├── 4b. 检查 ask rules → 需要询问？
 │ ├── 4c. tool.checkPermissions() → 工具自定义检查
 │ ├── 4d. 检查 permission mode → bypass/plan/default?
 │ ├── 4e. 检查 allow rules → 已授权？
 │ └── 4f. 默认 → 弹出权限确认对话框
 │
 ▼
5. PreToolUse Hooks: 执行前置钩子
 │ └── 用户自定义的 hook 脚本
 │
 ▼
6. 执行工具: tool.call(input, context, canUseTool, parentMessage, onProgress)
 │ └── 实际执行 glob 搜索
 │
 ▼
7. PostToolUse Hooks: 执行后置钩子
 │
 ▼
8. 结果序列化: tool.mapToolResultToToolResultBlockParam(output, toolUseID)
 │ └── 转换为 API 格式的 tool_result
 │
 ▼
9. 追加到消息列表，继续查询循环
```

## 2.6 ToolUseContext — 工具执行上下文

每个工具的 `call()` 方法都会收到一个 `ToolUseContext`，它包含了工具执行所需的一切：

```typescript
type ToolUseContext = {
  // ========== 配置 ==========
  options: {
    commands: Command[]; // 可用的 slash 命令
    tools: Tools; // 可用的工具列表
    mainLoopModel: string; // 当前使用的模型
    mcpClients: MCPServerConnection[]; // MCP 连接
    isNonInteractiveSession: boolean; // 是否非交互模式
    agentDefinitions: AgentDefinitionsResult; // Agent 定义
  };

  // ========== 状态管理 ==========
  getAppState(): AppState; // 获取全局状态
  setAppState(f): void; // 更新全局状态
  abortController: AbortController; // 中断控制器
  messages: Message[]; // 当前消息历史

  // ========== 文件缓存 ==========
  readFileState: FileStateCache; // 文件读取缓存（LRU）

  // ========== UI 回调 ==========
  setToolJSX?: SetToolJSXFn; // 设置工具 UI
  setInProgressToolUseIDs: (f) => void; // 标记进行中的工具
  setResponseLength: (f) => void; // 更新响应长度

  // ========== Agent 相关 ==========
  agentId?: AgentId; // 子 Agent ID
  agentType?: string; // Agent 类型名
};
```

**为什么用 `getAppState()` 而不是直接传 state？**
因为状态可能在工具执行过程中被其他并发操作修改。
每次调用 `getAppState()` 都能获取最新状态。

## 2.7 所有内置工具一览

| 工具名 | 文件 | 功能 | 只读 | 并发安全 |
|--------|------|------|------|----------|
| **Bash** | `BashTool/` | 执行 shell 命令 | | |
| **Read** | `FileReadTool/` | 读取文件内容 | | |
| **Edit** | `FileEditTool/` | 编辑文件 (diff) | | |
| **Write** | `FileWriteTool/` | 写入/创建文件 | | |
| **Glob** | `GlobTool/` | 文件名模式匹配 | | |
| **Grep** | `GrepTool/` | 文本内容搜索 | | |
| **Agent** | `AgentTool/` | 启动子 Agent | | |
| **WebSearch** | `WebSearchTool/` | 网页搜索 | | |
| **WebFetch** | `WebFetchTool/` | 获取网页内容 | | |
| **TodoWrite** | `TodoWriteTool/` | 管理 Todo 列表 | | |
| **Notebook** | `NotebookEditTool/` | 编辑 Jupyter | | |
| **Skill** | `SkillTool/` | 调用技能 | | |
| **AskUser** | `AskUserQuestionTool/` | 询问用户 | | |
| **TaskStop** | `TaskStopTool/` | 停止当前任务 | | |
| **TaskOutput** | `TaskOutputTool/` | 输出任务结果 | | |
| **SendMessage** | `SendMessageTool/` | Agent 间通信 | | |
| **ToolSearch** | `ToolSearchTool/` | 搜索可用工具 | | |
| **Brief** | `BriefTool/` | 切换简要模式 | | |

### 实验性工具 (Feature Flag 控制)

| 工具名 | Flag | 功能 |
|--------|------|------|
| SleepTool | `PROACTIVE`/`KAIROS` | 等待指定时间 |
| CronTools | `AGENT_TRIGGERS` | 定时任务管理 |
| WebBrowserTool | `WEB_BROWSER_TOOL` | 浏览器自动化 |
| MonitorTool | `MONITOR_TOOL` | 监控工具 |
| SnipTool | `HISTORY_SNIP` | 历史裁剪 |

## 2.8 ToolResult 与 contextModifier — 工具如何影响后续执行

工具的 `call()` 方法返回 `ToolResult<T>`，它不仅包含数据，还能修改后续执行上下文：

```typescript
// Tool.ts 中的定义（第 259-272 行）
export type ToolResult<T> = {
  data: T; // 工具输出数据
  newMessages?: ( // 注入额外消息
    | UserMessage | AssistantMessage
    | AttachmentMessage | SystemMessage
  )[];
  // 只对非并发安全的工具生效！
  contextModifier?: (context: ToolUseContext) => ToolUseContext;
  mcpMeta?: { // MCP 协议元数据透传
    _meta?: Record<string, unknown>;
    structuredContent?: Record<string, unknown>;
  };
};
```

**contextModifier 的妙用：**
```typescript
// 例如 FileEditTool 可以在编辑后更新文件缓存
return {
  data: editResult,
  contextModifier: (ctx) => ({
    ...ctx,
    readFileState: ctx.readFileState.set(filePath, newContent),
}),
};
```

**重要限制：** `contextModifier` 只对非并发安全的工具生效。
因为并发工具同时执行，无法保证 modifier 的应用顺序。
这是一个有意的设计约束，避免了竞态条件。

在 `toolOrchestration.ts` 中可以看到这个逻辑：
```typescript
// 串行工具：立即应用 contextModifier
if (update.newContext) {
  currentContext = update.newContext;
}

// 并行工具：延迟到批次结束后按顺序应用
if (update.contextModifier) {
  queuedContextModifiers[toolUseID].push(modifyContext);
}
// 批次结束后：
for (const block of blocks) {
  for (const modifier of queuedContextModifiers[block.id]) {
    currentContext = modifier(currentContext);
  }
}
```

## 2.9 Feature Flag 条件加载 — 源码中的 `feature()` 和 `require()`

tools.ts 中大量使用了条件加载模式，这是 Bun 的 tree-shaking 机制：

```typescript
// tools.ts 中的真实代码
import { feature } from 'bun:bundle';

// feature() 在编译时被替换为 true/false
// 当为 false 时，整个 require() 块会被 tree-shake 掉
const SleepTool = feature('PROACTIVE') || feature('KAIROS')
  ? require('./tools/SleepTool/SleepTool.js').SleepTool
  : null;

const WebBrowserTool = feature('WEB_BROWSER_TOOL')
  ? require('./tools/WebBrowserTool/WebBrowserTool.js').WebBrowserTool
  : null;

// Ant-only 工具（Anthropic 内部版本）
const REPLTool = process.env.USER_TYPE === 'ant'
  ? require('./tools/REPLTool/REPLTool.js').REPLTool
  : null;

// 懒加载打破循环依赖
const getTeamCreateTool = () => require('./tools/TeamCreateTool/TeamCreateTool.js').TeamCreateTool;
```

**三种条件加载模式：**

| 模式 | 用途 | 示例 |
|------|------|------|
| `feature('FLAG')` | 编译时 Feature Flag | SleepTool, WebBrowserTool |
| `process.env.USER_TYPE === 'ant'` | 运行时环境变量 | REPLTool, ConfigTool |
| `() => require(...)` | 懒加载函数 | TeamCreateTool (打破循环依赖) |

**已知的 Feature Flags（从源码中提取）：**

| Flag | 功能 | 状态 |
|------|------|------|
| `PROACTIVE` | 主动式 Agent（Sleep 等待） | 实验性 |
| `KAIROS` | 长期运行 Agent（定时任务、推送通知、GitHub Webhooks） | 实验性 |
| `AGENT_TRIGGERS` | Agent 触发器（Cron 定时任务） | 实验性 |
| `AGENT_TRIGGERS_REMOTE` | 远程 Agent 触发器 | 实验性 |
| `COORDINATOR_MODE` | 协调器模式（多 Worker） | 实验性 |
| `WEB_BROWSER_TOOL` | 浏览器自动化工具 | 实验性 |
| `MONITOR_TOOL` | 监控工具 | 实验性 |
| `HISTORY_SNIP` | 历史裁剪（SnipTool） | 实验性 |
| `CONTEXT_COLLAPSE` | 上下文折叠 | 实验性 |
| `REACTIVE_COMPACT` | 响应式压缩 | 实验性 |
| `TRANSCRIPT_CLASSIFIER` | 安全分类器（auto 模式） | 实验性 |
| `TOKEN_BUDGET` | Token 预算管理 | 实验性 |
| `CACHED_MICROCOMPACT` | 缓存微压缩 | 实验性 |
| `WORKFLOW_SCRIPTS` | 工作流脚本 | 实验性 |
| `UDS_INBOX` | Unix Domain Socket 通信 | 实验性 |
| `OVERFLOW_TEST_TOOL` | 溢出测试工具 | 测试 |
| `TERMINAL_PANEL` | 终端面板捕获 | 实验性 |
| `EXPERIMENTAL_SKILL_SEARCH` | 技能搜索 | 实验性 |
| `TEMPLATES` | 任务模板 | 实验性 |

## 2.10 工具的 Prompt Engineering 技巧

Claude Code 在工具描述中使用了大量 prompt engineering 技巧：

### 技巧1：引导 LLM 选择正确的工具
```typescript
// GlobTool prompt.ts
'When you are doing an open ended search that may require multiple\n'
'rounds of globbing and grepping, use the Agent tool instead'
// → 引导 LLM 在复杂搜索时使用 AgentTool
```

### 技巧2：在 Zod describe 中嵌入指令
```typescript
// GlobTool inputSchema
path: z.string().optional().describe(
  'IMPORTANT: Omit this field to use the default directory. '
  + 'DO NOT enter \"undefined\" or \"null\"'
)
// → 防止 LLM 传入 'undefined' 字符串
```

### 技巧3：结果中嵌入引导
```typescript
// GlobTool 截断提示
'(Results are truncated. Consider using a more specific path or pattern.)'
// → 引导 LLM 缩小搜索范围
```

### 技巧4：searchHint 优化 ToolSearch
```typescript
// 每个工具的 searchHint 帮助 ToolSearch 找到正确的工具
GlobTool: searchHint: 'find files by name pattern or wildcard'
GrepTool: searchHint: 'search file contents by regex pattern'
WebFetchTool: searchHint: 'download web page content from URL'
// → 当 ToolSearch 搜索 'download' 时，能找到 WebFetchTool
```

### 技巧5：isSearchOrReadCommand 控制 UI 折叠
```typescript
// 搜索/读取类操作在 UI 中自动折叠，减少视觉噪音
isSearchOrReadCommand() {
  return { isSearch: true, isRead: false, isList: false };
}
// BashTool 会根据命令内容动态判断：
// 'grep ...' → isSearch: true
// 'cat ...' → isRead: true
// 'ls ...' → isList: true
// 'rm ...' → 都是 false（不折叠）
```

## 2.11 工具的 maxResultSizeChars — 大结果处理

当工具输出超过 `maxResultSizeChars` 时，结果会被持久化到磁盘：

```
工具输出 (例如 200KB 的 grep 结果)
 │
 ▼
resultText.length > tool.maxResultSizeChars ?
 │
 ├── YES → 保存到 ~/.claude/projects/{cwd}/tool-results/{uuid}.txt
 │ 返回预览 + 文件路径给 Claude
 │ "[Full output saved to: /path/to/file]\n"
 │ "Use Read tool to view the full output if needed."
 │
 └── NO → 直接返回完整结果
```

**各工具的 maxResultSizeChars 设置：**

| 工具 | 限制 | 原因 |
|------|------|------|
| GlobTool | 100,000 | 文件列表可能很长 |
| GrepTool | 100,000 | 搜索结果可能很多 |
| BashTool | 30,000 | 命令输出可能很大 |
| FileReadTool | **Infinity** | 不持久化！避免 Read→file→Read 循环 |
| WebFetchTool | 50,000 | 网页内容可能很大 |

**为什么 FileReadTool 设为 Infinity？**
如果 Read 的结果被保存到文件，Claude 会用 Read 去读那个文件，
那个文件的结果又被保存...形成无限循环。所以 Read 工具自己控制输出大小。

**applyToolResultBudget — 聚合预算控制（query.ts 中）：**
```typescript
// 除了单个工具的 maxResultSizeChars，还有全局的聚合预算
// 在 query 循环的每次迭代中应用
messagesForQuery = await applyToolResultBudget(
  messagesForQuery,
  toolUseContext.contentReplacementState,
  persistReplacements ? records => void recordContentReplacement(...) : undefined,
  // 跳过 maxResultSizeChars 为 Infinity 的工具
  new Set(tools.filter(t => !Number.isFinite(t.maxResultSizeChars)).map(t => t.name)),
);
```

## 本章小结

| 概念 | 关键点 |
|------|--------|
| Tool 接口 | 28+ 个方法，覆盖执行、权限、UI、序列化 |
| buildTool() | 工厂函数，提供安全的默认值 (fail-closed) |
| 工具目录结构 | `XxxTool.ts` + `prompt.ts` + `UI.tsx` 三件套 |
| 工具注册 | `getAllBaseTools()` → `getTools()` → `assembleToolPool()` |
| 执行生命周期 | 查找 → 解析 → 验证 → 权限 → Hook → 执行 → Hook → 序列化 |
| ToolUseContext | 工具执行的完整上下文，包含状态、配置、回调 |
| ToolResult | 不仅返回数据，还能通过 contextModifier 修改后续上下文 |
| Feature Flags | 18+ 个实验性 flag 控制工具的条件加载 |
| Prompt Engineering | 5 种技巧引导 LLM 正确使用工具 |
| 大结果处理 | 超过 maxResultSizeChars 自动持久化到磁盘 |

### 下一章预告
第3章将深入 **权限与安全系统**——Claude Code 如何防止 AI 执行危险操作。

