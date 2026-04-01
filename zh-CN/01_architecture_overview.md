# 第1章：Claude Code 全局架构鸟瞰

## 学习目标
读完本章，你将理解：
1. Claude Code 是什么？它的核心价值是什么？
2. 一条用户消息从输入到输出，经历了哪些关键步骤？
3. 项目的目录结构和各模块的职责
4. 核心数据流和控制流

---

## 1.1 Claude Code 是什么？

Claude Code 是 Anthropic 官方开发的 **AI 编程助手 CLI 工具**。它的核心理念是：

> **让 Claude 模型直接操作你的文件系统和终端，像一个真正的编程搭档一样工作。**

### 技术栈概览

| 层级 | 技术 | 说明 |
|------|------|------|
| **语言** | TypeScript | 全项目使用 TS，严格类型 |
| **运行时** | Bun | 替代 Node.js，更快的启动和执行 |
| **终端 UI** | React + Ink | 用 React 组件渲染终端界面！ |
| **CLI 框架** | Commander.js | 命令行参数解析 |
| **LLM API** | Anthropic SDK | 调用 Claude 模型 |
| **扩展协议** | MCP (Model Context Protocol) | 标准化的工具扩展协议 |
| **构建** | Bun bundler | 带 feature flag 的条件编译 |

### 项目规模
```
文件数量: 1,900+
代码行数: 512,000+
核心源码: src/ 目录
```

## 1.2 核心架构图

```
┌─────────────────────────────────────────────────────────────────┐
│ 用户终端 (Terminal) │
└──────────────────────────────┬──────────────────────────────────┘
 │
 ▼
┌─────────────────────────────────────────────────────────────────┐
│ main.tsx (入口) │
│ ├── Commander.js 解析 CLI 参数 │
│ ├── 初始化: auth, config, MCP, plugins, skills │
│ └── 启动 React/Ink 渲染 → App.tsx │
└──────────────────────────────┬──────────────────────────────────┘
 │
 ▼
┌─────────────────────────────────────────────────────────────────┐
│ REPL 循环 (components/REPL.tsx) │
│ ├── 接收用户输入 │
│ ├── 处理 /slash 命令 │
│ └── 调用 QueryEngine │
└──────────────────────────────┬──────────────────────────────────┘
 │
 ▼
┌─────────────────────────────────────────────────────────────────┐
│ QueryEngine.ts (核心引擎) │
│ ├── 构建 System Prompt (context.ts + prompts.ts) │
│ ├── 调用 query() 循环 │
│ │ ├── 发送消息到 Claude API │
│ │ ├── 流式接收响应 │
│ │ ├── 解析 tool_use 块 │
│ │ ├── 权限检查 (permissions.ts) │
│ │ ├── 执行工具 (toolExecution.ts) │
│ │ ├── 将 tool_result 追加到消息 │
│ │ └── 循环直到 stop_reason = 'end_turn' │
│ └── 返回最终结果 │
└─────────────────────────────────────────────────────────────────┘
```

## 1.3 目录结构详解

```
src/
├── main.tsx # CLI 入口点 (785KB! 包含所有命令行参数定义)
├── QueryEngine.ts # 核心引擎 - 管理查询生命周期和会话状态
├── query.ts # 查询循环 - LLM 调用 → 工具执行 → 再调用的循环
├── Tool.ts # 工具类型定义 - 所有工具的接口契约
├── tools.ts # 工具注册表 - 组装和过滤工具池
├── context.ts # 上下文管理 - git status, CLAUDE.md 等
├── commands.ts # ⌨ /slash 命令注册
│
├── tools/ # 所有工具实现
│ ├── BashTool/ # 执行 shell 命令
│ ├── FileReadTool/ # 读取文件
│ ├── FileEditTool/ # 编辑文件 (diff/patch)
│ ├── FileWriteTool/ # 写入文件
│ ├── GlobTool/ # 文件名模式匹配
│ ├── GrepTool/ # 文本搜索
│ ├── AgentTool/ # 子 Agent 系统
│ ├── WebSearchTool/ # 网页搜索
│ ├── WebFetchTool/ # 获取网页内容
│ ├── MCPTool/ # MCP 工具包装器
│ └── ... # 更多工具
│
├── services/ # 外部服务集成
│ ├── api/ # Anthropic API 调用
│ ├── mcp/ # MCP 协议客户端
│ ├── compact/ # 上下文压缩 (auto-compact)
│ ├── analytics/ # 遥测和分析
│ └── tools/ # 工具执行引擎
│
├── hooks/ # 🪝 Hook 系统
│ ├── useCanUseTool.ts # 工具权限 hook
│ └── toolPermission/ # 权限处理器
│
├── components/ # React/Ink UI 组件
│ ├── App.tsx # 根组件
│ ├── REPL.tsx # REPL 循环组件
│ ├── Messages.tsx # 消息渲染
│ └── PromptInput/ # 输入框
│
├── state/ # 状态管理
│ ├── AppState.tsx # 应用状态类型
│ └── store.ts # 状态存储
│
├── coordinator/ # 多 Agent 协调
├── bridge/ # IDE 集成 (VS Code 等)
├── skills/ # 可复用工作流
├── plugins/ # 插件系统
├── memdir/ # 记忆系统
├── entrypoints/ # 不同启动模式
└── utils/ # 工具函数
 ├── permissions/ # 权限系统核心
 ├── model/ # 模型管理
 ├── messages.ts # 消息处理
 └── ... # 更多工具函数
```

## 1.4 核心数据流：一条消息的完整旅程

让我们追踪用户输入 `"帮我读取 package.json"` 的完整流程：

```
Step 1: 用户输入
 └── REPL.tsx 接收输入文本

Step 2: 消息构建
 └── processUserInput() 将文本转为 Message 对象
 ├── 检查是否是 /slash 命令
 ├── 处理附件 (图片、文件等)
 └── 构建 UserMessage { role: 'user', content: '帮我读取 package.json' }

Step 3: QueryEngine.submitMessage()
 ├── 获取 System Prompt
 │ ├── 默认系统提示词 (constants/prompts.ts)
 │ ├── 用户上下文: CLAUDE.md 内容 + 当前日期
 │ └── 系统上下文: git status + branch 信息
 ├── 注册工具列表 (tools.ts → getAllBaseTools())
 └── 调用 query() 进入查询循环

Step 4: query() 查询循环
 ├── 4a. 调用 Claude API
 │ ├── normalizeMessagesForAPI() 格式化消息
 │ ├── prependUserContext() 注入用户上下文
 │ ├── appendSystemContext() 注入系统上下文
 │ └── 发送 HTTP 请求到 Anthropic API
 │
 ├── 4b. 流式接收响应
 │ ├── message_start → 初始化
 │ ├── content_block_start → 开始接收内容块
 │ │ ├── type: 'text' → 文本响应
 │ │ └── type: 'tool_use' → 工具调用请求！
 │ │ { name: 'Read', input: { file_path: 'package.json' } }
 │ ├── content_block_delta → 增量内容
 │ └── message_stop → 消息结束
 │
 ├── 4c. 工具执行 (当 stop_reason = 'tool_use')
 │ ├── 权限检查: hasPermissionsToUseTool()
 │ │ ├── 检查 deny rules → 是否被禁止？
 │ │ ├── 检查 allow rules → 是否已授权？
 │ │ ├── 检查 permission mode → bypass/plan/default?
 │ │ └── 如果需要 → 弹出权限确认对话框
 │ ├── 输入验证: tool.validateInput()
 │ ├── 执行工具: tool.call()
 │ │ └── FileReadTool.call({ file_path: 'package.json' })
 │ │ → 读取文件内容并返回
 │ └── 构建 tool_result 消息
 │
 └── 4d. 继续循环
 ├── 将 tool_result 追加到消息列表
 ├── 再次调用 Claude API (带上工具结果)
 ├── Claude 生成最终文本响应
 └── stop_reason = 'end_turn' → 循环结束

Step 5: 结果返回
 ├── 渲染 Claude 的文本响应到终端
 ├── 记录会话到 transcript
 └── 等待下一次用户输入
```

## 1.5 关键文件速览

### main.tsx — CLI 入口 (785KB)

这是整个项目最大的文件！它做了以下事情：

```typescript
// 1. 性能优化：在所有 import 之前启动并行预加载
profileCheckpoint('main_tsx_entry'); // 标记启动时间
startMdmRawRead(); // 并行读取 MDM 配置
startKeychainPrefetch(); // 并行读取 keychain 凭证

// 2. 使用 Commander.js 定义所有 CLI 参数
const program = new CommanderCommand()
  .option('-p, --print', '非交互模式')
  .option('--model <model>', '指定模型')
  .option('--permission-mode <mode>', '权限模式');
// ... 几十个参数

// 3. 初始化各子系统
await init(); // 认证、配置、遥测
await initializeGrowthBook(); // Feature flags
await initBundledSkills(); // 内置技能
await initBuiltinPlugins(); // 内置插件

// 4. 启动 REPL 或 print 模式
launchRepl(options); // 交互模式
// 或
ask({ prompt, tools, ... }); // 非交互模式
```

### QueryEngine.ts — 核心引擎 (45KB)

```typescript
export class QueryEngine {
  // 每个会话一个实例
  private mutableMessages: Message[]; // 消息历史
  private abortController: AbortController; // 中断控制
  private totalUsage: NonNullableUsage; // Token 使用量追踪

  // 核心方法：提交消息并获取响应流
  async *submitMessage(prompt): AsyncGenerator<SDKMessage> {
    // 1. 构建系统提示词
    const { defaultSystemPrompt, userContext, systemContext } =
      await fetchSystemPromptParts({ tools, mainLoopModel, ... });

    // 2. 处理用户输入 (slash 命令等)
    const { messages, shouldQuery } = await processUserInput(...);

    // 3. 进入查询循环
    for await (const message of query({ messages, systemPrompt, ... })) {
      // 处理各种消息类型
      switch (message.type) {
        case 'assistant': yield* normalizeMessage(message); break;
        case 'user': yield* normalizeMessage(message); break;
        case 'stream_event': /* 流式事件 */ break;
        case 'attachment': /* 附件/结构化输出 */ break;
      }
    }

    // 4. 返回最终结果
    yield { type: 'result', subtype: 'success', ... };
  }
}
```

## 1.6 Feature Flags 系统

Claude Code 使用 `bun:bundle` 的 `feature()` 函数实现**编译时条件编译**：

```typescript
import { feature } from 'bun:bundle';

// 编译时决定是否包含代码
const coordinatorModeModule = feature('COORDINATOR_MODE')
  ? require('./coordinator/coordinatorMode.js')
  : null;

const SleepTool = feature('PROACTIVE') || feature('KAIROS')
  ? require('./tools/SleepTool/SleepTool.js').SleepTool
  : null;
```

### 已知的 Feature Flags

| Flag | 功能 | 状态 |
|------|------|------|
| `COORDINATOR_MODE` | 多 Agent 协调模式 | 实验性 |
| `KAIROS` | 助手模式 (长期运行) | 实验性 |
| `PROACTIVE` | 主动式 Agent | 实验性 |
| `AGENT_TRIGGERS` | Agent 触发器 (Cron) | 实验性 |
| `HISTORY_SNIP` | 历史裁剪 | 实验性 |
| `WEB_BROWSER_TOOL` | 浏览器工具 | 实验性 |
| `CONTEXT_COLLAPSE` | 上下文折叠 | 实验性 |
| `TOKEN_BUDGET` | Token 预算管理 | 实验性 |
| `REACTIVE_COMPACT` | 响应式压缩 | 实验性 |
| `TRANSCRIPT_CLASSIFIER` | 自动模式分类器 | 实验性 |

这些 flags 标记了 Anthropic 正在探索的前沿方向！

## 1.7 消息类型系统

Claude Code 的消息系统是整个架构的骨架：

```typescript
// 核心消息类型 (src/types/message.ts)
type Message =
  | UserMessage // 用户输入
  | AssistantMessage // Claude 响应
  | SystemMessage // 系统消息 (compact_boundary, api_error 等)
  | ProgressMessage // 工具执行进度
  | AttachmentMessage // 附件 (图片、文件、结构化输出)
  | StreamEvent; // 流式事件 (message_start, content_block_delta 等)

// UserMessage 的 content 可以是：
// - string: 纯文本
// - ContentBlockParam[]: 包含 text, image, tool_result 等

// AssistantMessage 的 content 包含：
// - { type: 'text', text: '...' } // 文本响应
// - { type: 'tool_use', name, input } // 工具调用
// - { type: 'thinking', thinking: '...' } // 思考过程
```

### 消息流转示意

```
User: "帮我读取 package.json"
 ↓
Assistant: [tool_use: { name: 'Read', input: { file_path: 'package.json' } }]
 ↓
User: [tool_result: { content: '{"name": "claude-code", ...}' }]
 ↓
Assistant: [text: "package.json 的内容如下：..."]
```

## 1.8 关键设计模式

### 1. AsyncGenerator 模式
整个查询循环使用 `async function*` (异步生成器)，这是最核心的设计模式：

```typescript
// QueryEngine.ts
async * submitMessage(prompt): AsyncGenerator < SDKMessage > {
  for await(const message of query({ ... })) {
  yield message; // 逐条产出消息
}
}

// query.ts
async function* query(params): AsyncGenerator<Message> {
  while (true) {
    // 调用 API
    for await (const event of apiStream) {
      yield event; // 流式产出
    }
    // 执行工具
    yield* runTools(...);
    // 判断是否继续
    if (shouldStop) break;
  }
}
```

**为什么用 AsyncGenerator？**
- 支持流式输出（打字机效果）
- 内存友好（不需要缓存所有消息）
- 支持中断（abort）
- 自然表达"循环直到完成"的语义

### 2. buildTool 工厂模式
所有工具通过 `buildTool()` 创建，自动填充默认值：

```typescript
export const GlobTool = buildTool({
  name: 'Glob',
  inputSchema: z.object({ pattern: z.string() }),
  async call(input, context) { ... },;
 // 以下由 buildTool 提供默认值：
 // isEnabled: () => true
 // isReadOnly: () => false
 // isConcurrencySafe: () => false
 // checkPermissions: () => ({ behavior: 'allow' })
});
```

### 3. Dead Code Elimination
通过 `feature()` + `require()` 实现编译时代码消除：

```typescript
// 如果 COORDINATOR_MODE 为 false，整个 require 和相关代码
// 在编译时被完全移除，不会出现在最终 bundle 中
const module = feature('COORDINATOR_MODE') 
  ? require('./coordinator/coordinatorMode.js') 
  : null;
```

## 1.9 context.ts — 上下文管理

每次对话开始时，Claude Code 会收集两类上下文：

### getUserContext() — 用户上下文
```typescript
export const getUserContext = memoize(async () => {
  // 1. 读取 CLAUDE.md 文件 (项目级 + 用户级)
  const claudeMd = getClaudeMds(await getMemoryFiles());
 
  // 2. 当前日期
  const currentDate = `Today's date is ${getLocalISODate()}.`;
 
  return { claudeMd, currentDate };
});
```

### getSystemContext() — 系统上下文
```typescript
export const getSystemContext = memoize(async () => {
  // 1. Git 状态 (branch, status, recent commits)
  const gitStatus = await getGitStatus();
  // 包含：当前分支、主分支、git status --short、最近5条commit
 
  return { gitStatus };
});
```

**关键点：** 这两个函数都使用 `memoize`，意味着在一次会话中只计算一次。
这就是为什么 git status 的注释说 "this status is a snapshot in time"。

## 本章小结

| 概念 | 关键文件 | 一句话总结 |
|------|----------|------------|
| CLI 入口 | `main.tsx` | Commander.js 解析参数，初始化所有子系统 |
| 核心引擎 | `QueryEngine.ts` | 管理会话状态，驱动查询循环 |
| 查询循环 | `query.ts` | LLM 调用 → 工具执行 → 再调用的核心循环 |
| 工具类型 | `Tool.ts` | 定义所有工具必须实现的接口 |
| 工具注册 | `tools.ts` | 组装、过滤、合并工具池 |
| 上下文 | `context.ts` | 收集 CLAUDE.md 和 git 信息 |

### 下一章预告
第2章将深入 **工具系统**——Claude 的"手和脚"，从最简单的 GlobTool 开始，
逐步理解工具如何定义、注册、执行。

