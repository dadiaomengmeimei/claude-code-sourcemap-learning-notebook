
# 🧠 Claude Code Sourcemap Learning Notebook

[![GitHub](https://img.shields.io/badge/GitHub-dadiaomengmeimei-blue?logo=github)](https://github.com/dadiaomengmeimei/claude-code-sourcemap-learning-notebook)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)

> 深入解析 Anthropic Claude Code CLI 工具的内部架构，从入口到工具系统、权限安全、查询循环、多 Agent 协作、MCP 扩展协议——用 Jupyter Notebook 带你逐层拆解 50 万行 TypeScript 代码。

## 📖 项目简介

[Claude Code](https://docs.anthropic.com/en/docs/claude-code) 是 Anthropic 开发的 AI 编程助手 CLI 工具，允许开发者从终端直接与 Claude AI 交互，执行代码编辑、文件搜索、Shell 命令、代码审查等软件工程任务。

本仓库是一套**结构化的学习笔记**，以 Jupyter Notebook 的形式，对 Claude Code 的核心源码进行深入浅出的解析。适合以下读者：

- 🔬 **AI 工具研究者** — 想了解工业级 AI Agent 系统的架构设计
- 🛡️ **安全研究者** — 想研究 AI 工具的权限控制和安全机制
- 🏗️ **架构师** — 想学习大规模 TypeScript 项目的工程实践
- 🤖 **Agent 开发者** — 想了解多 Agent 协作和工具系统的设计模式

## 🏗️ Claude Code 技术概况

| 类别 | 详情 |
|------|------|
| **语言** | TypeScript (strict mode) |
| **运行时** | Bun |
| **终端 UI** | React + Ink |
| **CLI 解析** | Commander.js |
| **Schema 验证** | Zod v4 |
| **代码搜索** | ripgrep |
| **扩展协议** | MCP (Model Context Protocol) |
| **API** | Anthropic SDK |
| **代码规模** | ~1,900 文件, 512,000+ 行代码 |

## 📚 学习路线

```
                    ┌─────────────────┐
                    │  00 总目录索引   │
                    └────────┬────────┘
                             │
              ┌──────────────┼──────────────┐
              │              │              │
     ┌────────▼───────┐ ┌───▼────────┐ ┌──▼──────────┐
     │ 01 全局架构    │ │ 02 工具系统 │ │ 03 权限安全  │
     │ (入口+状态)    │ │ (核心抽象)  │ │ (纵深防御)   │
     └────────┬───────┘ └───┬────────┘ └──┬──────────┘
              │              │              │
              └──────────────┼──────────────┘
                             │
                    ┌────────▼────────┐
                    │ 04 查询循环     │
                    │ (系统心脏)      │
                    └────────┬────────┘
                             │
                    ┌────────┼────────┐
                    │                 │
           ┌───────▼──────┐ ┌────────▼───────┐
           │ 05 多Agent   │ │ 06 MCP扩展     │
           │ (协作架构)   │ │ (开放生态)     │
           └──────────────┘ └────────────────┘
```

### 章节详情

| # | 文件 | 主题 | 核心源码文件 | 预计时间 |
|---|------|------|-------------|----------|
| 00 | [00_index.ipynb](00_index.ipynb) | 📚 总目录与架构全景图 | — | 10min |
| 01 | [01_architecture_overview.ipynb](01_architecture_overview.ipynb) | 🏗️ 全局架构鸟瞰 | `main.tsx`, `App.tsx`, `QueryEngine.ts` | 30min |
| 02 | [02_tool_system.ipynb](02_tool_system.ipynb) | 🔧 工具系统深度解析 | `Tool.ts`, `tools.ts`, `GlobTool.ts` | 45min |
| 03 | [03_permission_security.ipynb](03_permission_security.ipynb) | 🛡️ 权限与安全系统 | `permissions.ts`, `filesystem.ts` | 45min |
| 04 | [04_query_loop_api.ipynb](04_query_loop_api.ipynb) | 🔄 查询循环与 API 交互 | `query.ts`, `StreamingToolExecutor.ts` | 40min |
| 05 | [05_multi_agent_system.ipynb](05_multi_agent_system.ipynb) | 🤖 多 Agent 系统 | `AgentTool.tsx`, `runAgent.ts` | 40min |
| 06 | [06_mcp_extensions.ipynb](06_mcp_extensions.ipynb) | 🔌 MCP 协议与扩展 | `mcp/client.ts`, `skills/`, `plugins/` | 30min |

**总计约 4 小时**，可以系统性地理解 Claude Code 的核心架构。

## 🎯 推荐学习路径

### 🚀 快速入门 (2 小时)
```
00 → 01 → 02 → 04
```
理解：入口文件 → 工具定义 → 查询循环

### 🛡️ 安全研究 (1.5 小时)
```
00 → 03 → 02(权限部分)
```
理解：权限模型 → 命令安全分析 → 沙箱机制

### 🤖 Agent 研究 (2 小时)
```
00 → 05 → 04 → 06
```
理解：Agent 架构 → 查询循环 → MCP 扩展

### 📖 完整学习 (4 小时)
```
00 → 01 → 02 → 03 → 04 → 05 → 06
```

## 🗺️ 架构全景图

```
┌─────────────────────────────────────────────────────────────┐
│                     Claude Code 架构                        │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌──────────┐    ┌──────────────┐    ┌──────────────────┐  │
│  │ CLI/REPL │    │ Bridge/IDE   │    │ SDK/Headless     │  │
│  │ (Ink UI) │    │ (VS Code)    │    │ (Programmatic)   │  │
│  └────┬─────┘    └──────┬───────┘    └────────┬─────────┘  │
│       │                 │                      │            │
│       └─────────────────┼──────────────────────┘            │
│                         │                                   │
│                  ┌──────▼──────┐                            │
│                  │  App State  │                             │
│                  └──────┬──────┘                            │
│                         │                                   │
│                  ┌──────▼──────┐                            │
│                  │ QueryEngine │  ← 系统心脏                │
│                  │  query()    │                            │
│                  └──┬───┬──┬──┘                            │
│                     │   │  │                                │
│          ┌──────────┘   │  └──────────┐                    │
│          │              │              │                    │
│   ┌──────▼──────┐ ┌────▼─────┐ ┌──────▼──────┐            │
│   │ Tool System │ │ Claude   │ │ Permission  │            │
│   │ 40+ 工具    │ │ API      │ │ System      │            │
│   └──────┬──────┘ └──────────┘ │ 5层纵深防御  │            │
│          │                      └─────────────┘            │
│   ┌──────┴──────────────────────────────┐                  │
│   │  Built-in Tools    MCP Tools        │                  │
│   │  ├── Bash          ├── mcp__db__*   │                  │
│   │  ├── Read/Edit     ├── mcp__api__*  │                  │
│   │  ├── Agent         └── ...          │                  │
│   │  ├── WebSearch                      │                  │
│   │  └── 30+ more                       │                  │
│   └─────────────────────────────────────┘                  │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

## 📁 关键源码文件速查

| 文件 | 大小 | 功能 |
|------|------|------|
| `src/main.tsx` | 785KB | CLI 入口，Commander.js 命令定义 |
| `src/Tool.ts` | 28KB | **工具接口定义** — 所有工具的"宪法" |
| `src/tools.ts` | 16KB | **工具注册表** — 决定哪些工具可用 |
| `src/query.ts` | 67KB | **核心查询循环** — 系统的心脏 |
| `src/QueryEngine.ts` | 45KB | 查询引擎封装 |
| `src/utils/permissions/permissions.ts` | 51KB | **权限检查核心** |
| `src/utils/permissions/filesystem.ts` | 61KB | 文件系统权限 |
| `src/tools/BashTool/bashSecurity.ts` | 100KB | 命令安全分析 |
| `src/tools/AgentTool/AgentTool.tsx` | ~50KB | Agent 工具实现 |
| `src/tools/AgentTool/runAgent.ts` | 34KB | Agent 生命周期 |
| `src/services/mcp/client.ts` | 116KB | MCP 客户端 |
| `src/services/tools/StreamingToolExecutor.ts` | 17KB | 流式工具执行器 |
| `src/services/tools/toolOrchestration.ts` | 5KB | 工具编排（并发/串行） |

## 🔑 核心设计理念

### 1. Fail-Closed 安全优先
```typescript
// 所有工具默认假设不安全
const TOOL_DEFAULTS = {
  isConcurrencySafe: () => false,  // 不确定就不并发
  isReadOnly: () => false,          // 不确定就假设会写入
};
```

### 2. 纵深防御 (Defense in Depth)
```
Layer 1: Permission Rules (deny > ask > allow)
  └── Layer 2: Permission Mode (default/plan/bypass/auto)
      └── Layer 3: Tool-specific checks (BashTool security)
          └── Layer 4: Path safety checks (filesystem)
              └── Layer 5: Sandbox (macOS Seatbelt)
```

### 3. 流式工具执行
```
Claude API 流式返回 tool_use 块
  → 接收到完整输入后立即开始执行
  → 并发安全的工具可并行
  → 结果按顺序返回
```

### 4. Prompt Cache 优化
```
工具列表排序 → 顺序稳定 → 最大化 cache 命中率
内置工具在前 → MCP 工具在后 → 前缀不变
```

## 🔬 每章核心发现

| 章节 | 最有价值的发现 |
|------|---------------|
| 01 架构 | React/Ink 终端 UI + 函数式状态管理，CLI 也能用 React |
| 02 工具 | `buildTool()` 工厂模式 + Zod schema 即 prompt，18+ Feature Flags |
| 03 安全 | 5 层纵深防御，bypass 模式也有安全底线（.git/.claude 不可绕过） |
| 04 查询 | `StreamingToolExecutor` 流式并发执行，Bash 错误级联取消兄弟工具 |
| 05 Agent | Explore Agent 省略 CLAUDE.md 节省 token，8 步清理防资源泄露 |
| 06 MCP | `mcp__server__tool` 命名约定，ToolSearch 延迟加载减少初始 token |

## 🛠️ 如何使用

### 方式一：Jupyter Notebook
```bash
# 安装 Jupyter
pip install jupyter

# 打开笔记
cd learning-notebooks
jupyter notebook
```

### 方式二：VS Code
直接用 VS Code 打开 `.ipynb` 文件，安装 Jupyter 扩展即可。

### 方式三：GitHub 在线浏览
GitHub 原生支持 `.ipynb` 文件的渲染，直接点击文件即可阅读。

## ⚠️ 免责声明

本项目仅用于**教育和研究目的**。源码来自通过 npm 包 source map 文件公开暴露的 Claude Code TypeScript 源码。本项目不包含任何原始源码文件，仅包含对架构和设计模式的分析笔记。

## 📄 License

本学习笔记以 MIT License 发布。请注意，被分析的 Claude Code 源码版权归 Anthropic 所有。

## 🤝 贡献

欢迎提交 Issue 和 PR！如果你发现了有趣的设计模式或安全机制，欢迎补充到笔记中。

**贡献方式：**
1. Fork 本仓库
2. 创建你的分支 (`git checkout -b feature/amazing-discovery`)
3. 提交修改 (`git commit -m 'Add: 发现了 XXX 的有趣设计'`)
4. 推送到分支 (`git push origin feature/amazing-discovery`)
5. 提交 Pull Request

## 🔗 相关链接

- 📦 [Claude Code 官方文档](https://docs.anthropic.com/en/docs/claude-code)
- 🏠 [Anthropic 官网](https://www.anthropic.com/)
- 📋 [MCP 协议规范](https://modelcontextprotocol.io/)
- 🐙 [本项目 GitHub](https://github.com/dadiaomengmeimei/claude-code-sourcemap-learning-notebook)

---

**⭐ 如果这个项目对你有帮助，请给一个 Star！**

**📢 关注作者 [@dadiaomengmeimei](https://github.com/dadiaomengmeimei) 获取更多 AI 工具源码分析！**
