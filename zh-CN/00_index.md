# Claude Code 源码学习指南 — 总目录

## 项目概况
- **项目**: Claude Code (Anthropic 官方 AI 编程助手)
- **规模**: 1,900+ 文件, 512,000+ 行 TypeScript 代码
- **技术栈**: TypeScript + React/Ink (终端 UI) + Bun (运行时)
- **架构**: Agent-based, Tool-augmented LLM 系统

---

## 学习路线

| 章节 | 文件 | 主题 | 核心文件 | 预计时间 |
|------|------|------|----------|----------|
| 01 | [01_architecture_overview.md](01_architecture_overview.md) | 全局架构鸟瞰 | main.tsx, App.tsx, QueryEngine.ts | 30min |
| 02 | [02_tool_system.md](02_tool_system.md) | 工具系统 | Tool.ts, tools.ts, GlobTool.ts | 45min |
| 03 | [03_permission_security.md](03_permission_security.md) | 权限与安全 | permissions.ts, bashSecurity.ts | 45min |
| 04 | [04_query_loop_api.md](04_query_loop_api.md) | 查询循环与 API (深度版) | query.ts, StreamingToolExecutor.ts | 50min |
| 04b | [04b_context_management.md](04b_context_management.md) | 上下文管理：召回、压缩与渐进式披露 | context.ts, autoCompact.ts, toolSearch.ts | 40min |
| 05 | [05_multi_agent_system.md](05_multi_agent_system.md) | 多 Agent 系统 + Coordinator (深度版) | AgentTool.tsx, runAgent.ts, coordinatorMode.ts | 50min |
| 06 | [06_mcp_extensions.md](06_mcp_extensions.md) | MCP、Skills 与扩展系统 | mcp/client.ts, skills/, bundledSkills.ts | 45min |
| 07 | [07_prompt_engineering.md](07_prompt_engineering.md) | Prompt Engineering 深度解析 | prompts.ts, BashTool/prompt.ts | 60min |
| 08 | [08_voice_buddy.md](08_voice_buddy.md) | Voice 模式与 Buddy 系统 | voiceStreamSTT.ts, useVoice.ts, buddy/ | 30min |

**总计约 6 小时** 可以系统性理解 Claude Code 的核心架构。
---

## 架构全景图

```
┌─────────────────────────────────────────────────────────────┐
│ Claude Code 架构 │
├─────────────────────────────────────────────────────────────┤
│ │
│ ┌──────────┐ ┌──────────────┐ ┌──────────────────┐ │
│ │ CLI/REPL │ │ Bridge/IDE │ │ SDK/Headless │ │
│ │ (Ink UI) │ │ (VS Code) │ │ (Programmatic) │ │
│ └────┬─────┘ └──────┬───────┘ └────────┬─────────┘ │
│ │ │ │ │
│ └─────────────────┼──────────────────────┘ │
│ │ │
│ ┌──────▼──────┐ │
│ │ App State │ (Ch.01) │
│ └──────┬──────┘ │
│ │ │
│ ┌──────▼──────┐ │
│ │ QueryEngine │ (Ch.04) │
│ │ query() │ │
│ └──┬───┬──┬──┘ │
│ │ │ │ │
│ ┌──────────┘ │ └──────────┐ │
│ │ │ │ │
│ ┌──────▼──────┐ ┌────▼─────┐ ┌──────▼──────┐ │
│ │ Tool System │ │ Claude │ │ Permission │ │
│ │ (Ch.02) │ │ API │ │ System │ │
│ └──────┬──────┘ └──────────┘ │ (Ch.03) │ │
│ │ └─────────────┘ │
│ ┌──────┴──────────────────────────────┐ │
│ │ │ │
│ │ Built-in Tools MCP Tools │ │
│ │ ├── Bash ├── mcp__db__* │ │
│ │ ├── Read ├── mcp__api__* │ │
│ │ ├── Edit └── ... │ │
│ │ ├── Agent (Ch.05) │ │
│ │ └── ... MCP (Ch.06) │ │
│ └─────────────────────────────────────┘ │
│ │
└─────────────────────────────────────────────────────────────┘
```

---

## 推荐学习策略

### 初学者路线 (2小时)
1. 第1章 → 第2章 → 第4章
2. 理解：入口 → 工具定义 → 查询循环

### 安全研究路线 (1.5小时)
1. 第3章 → 第2章(权限部分)
2. 理解：权限模型 → 命令安全 → 沙箱

### Agent 研究路线 (2.5小时)
1. 第5章 → 第4章 → 第6章
2. 理解：Agent 架构 + Coordinator → 查询循环 → MCP/Skills 扩展

### Prompt 研究路线 (1小时)
1. 第7章
2. 理解：系统提示词 → 工具提示词 → 压缩提示词 → 缓存优化

### 特色功能路线 (30分钟)
1. 第8章
2. 理解：Voice hold-to-talk → WebSocket STT → Buddy 确定性生成

### 完整路线 (5.5小时)
1. 按顺序 01 → 08 全部阅读

---

## 关键文件速查表

| 文件 | 大小 | 功能 |
|------|------|------|
| `src/main.tsx` | 785KB | CLI 入口，Commander.js 命令定义 |
| `src/Tool.ts` | 28KB | 工具接口定义 |
| `src/tools.ts` | 16KB | 工具注册表 |
| `src/query.ts` | 67KB | 核心查询循环 |
| `src/QueryEngine.ts` | 45KB | 查询引擎封装 |
| `src/context.ts` | 6KB | 上下文管理 |
| `src/tools/BashTool/bashSecurity.ts` | 100KB | 命令安全分析 |
| `src/tools/BashTool/bashPermissions.ts` | 96KB | 命令权限检查 |
| `src/tools/AgentTool/AgentTool.tsx` | ~50KB | Agent 工具实现 |
| `src/tools/AgentTool/runAgent.ts` | 34KB | Agent 生命周期 |
| `src/services/mcp/client.ts` | 116KB | MCP 客户端 |
| `src/services/mcp/config.ts` | ~20KB | MCP 配置管理 |

