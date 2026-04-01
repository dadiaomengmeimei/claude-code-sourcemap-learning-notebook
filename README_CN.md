# Claude Code 源码解析学习笔记

[![GitHub](https://img.shields.io/badge/GitHub-dadiaomengmeimei-blue?logo=github)](https://github.com/dadiaomengmeimei/claude-code-sourcemap-learning-notebook)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)

> 🌐 [English Version](README.md)

**逆向工程 Claude Code 的 51 万行 TypeScript 源码 — 提炼架构设计、决策逻辑与 11 个可迁移的工程模式。**

不只是代码走读，而是架构分析：每个设计选择都解释了*为什么*这样做，并与替代方案对比。

## 你将学到什么

- 🔄 **查询循环** — 流式工具执行、读写锁并发、三层 AbortController 级联取消
- 🛠️ **工具系统** — `buildTool()` 工厂模式 + Zod schema 即 prompt，40+ 工具，18+ Feature Flags
- 🔒 **5 层安全纵深** — 权限规则 → 模式 → 工具检查 → 路径安全 → macOS Seatbelt 沙箱
- 🤖 **多 Agent** — Coordinator 编排、默认隔离显式共享、Cache 友好的 Fork
- 🔌 **MCP & Skills** — 三层 Skill 架构、bundled skill 懒提取、`/simplify` 三 Agent 并行审查
- 🧠 **Prompt 工程** — 7 大类 40+ prompt 文件、静态/动态分离缓存优化
- 🎙️ **Voice & Buddy** — 录音先行连接后补消除延迟、确定性生成

## 章节目录

| # | 主题 | 核心源码文件 | 时间 |
|---|------|-------------|------|
| [00](zh-CN/00_index.md) | 总目录与架构全景图 | — | 10min |
| [01](zh-CN/01_architecture_overview.md) | 全局架构鸟瞰 | `main.tsx`, `App.tsx`, `QueryEngine.ts` | 30min |
| [02](zh-CN/02_tool_system.md) | 工具系统深度解析 | `Tool.ts`, `tools.ts`, `GlobTool.ts` | 45min |
| [03](zh-CN/03_permission_security.md) | 权限与安全系统 | `permissions.ts`, `filesystem.ts` | 45min |
| [04](zh-CN/04_query_loop_api.md) | 查询循环与 API ⭐ | `query.ts`, `StreamingToolExecutor.ts` | 50min |
| [05](zh-CN/05_multi_agent_system.md) | 多 Agent + Coordinator ⭐ | `AgentTool.tsx`, `runAgent.ts` | 50min |
| [06](zh-CN/06_mcp_extensions.md) | MCP、Skills 与扩展 | `mcp/client.ts`, `skills/` | 45min |
| [07](zh-CN/07_prompt_engineering.md) | Prompt Engineering | `prompts.ts`, `BashTool/prompt.ts` | 60min |
| [08](zh-CN/08_voice_buddy.md) | Voice 模式与 Buddy | `voiceStreamSTT.ts`, `buddy/` | 30min |

**总计约 5.5 小时**，系统性理解 Claude Code 核心架构。

> 课程内容提供 [中文 (zh-CN/)](zh-CN/) 和 [英文 (en/)](en/) 两个版本。

## 学习路径

| 路径 | 章节 | 时间 |
|------|------|------|
| **快速入门** | 00 → 01 → 02 → 04 | 2h |
| **安全研究** | 00 → 03 → 02 | 1.5h |
| **Agent 研究** | 00 → 05 → 04 → 06 | 2.5h |
| **Prompt 研究** | 00 → 07 | 1h |
| **完整学习** | 00 → 01 → ... → 08 | 5.5h |

## 每章核心发现

| 章节 | 最有价值的发现 |
|------|---------------|
| 01 | React/Ink 终端 UI — CLI 也能用 React |
| 02 | `buildTool()` 工厂模式 + Zod schema 兼做 prompt |
| 03 | 5 层纵深防御，bypass 模式也有安全底线（.git/.claude 不可绕过） |
| 04 | `StreamingToolExecutor` 并发执行，5 层压缩管线 |
| 05 | Coordinator 星型拓扑，Fork prompt cache 共享，10 步 finally 清理 |
| 06 | Skills 三层架构，`/simplify` 三 Agent 并行审查 |
| 07 | 40+ prompt 文件静态/动态分离，Meta-Prompt 生成 Agent |
| 08 | 录音先行连接后补消除延迟，Buddy Bones 不持久化防伪造 |

## 11 个可迁移设计模式

提炼自第 04、05 章 — 可应用于任何 Agent 系统：

**查询循环：** 乐观恢复 · 分层降级 · 状态机+转换日志 · 读写锁并发 · 不可变配置快照 · 层级取消

**多 Agent：** 基于能力的安全 · Cache 友好 Fork · 确定性清理 · 星型拓扑编排 · 单调权限收窄

> 每个模式在章节笔记中都包含 Claude Code 实现细节和通用应用场景（DB、K8s、微服务等）。

## 姊妹项目：nano-claude-code

> **1,646 行 TypeScript · 15 个文件 · 4 个依赖**

[nano-claude-code](https://github.com/dadiaomengmeimei/nano-claude-code) 将 Claude Code 的核心从零重写为一个极简、可 hack 的实现。

| | Claude Code | nano-claude-code |
|---|---|---|
| 文件数 | ~1,900 | **15** |
| 代码行数 | 512,000+ | **1,646** |
| 依赖 | 50+ | **4** |
| 工具 | 40+ | **6** |

**读笔记理解架构，跑 nano-claude-code 动手实践。**

👉 [github.com/dadiaomengmeimei/nano-claude-code](https://github.com/dadiaomengmeimei/nano-claude-code)

## 相关链接

- [Claude Code 官方文档](https://docs.anthropic.com/en/docs/claude-code)
- [MCP 协议规范](https://modelcontextprotocol.io/)
- [本项目 GitHub](https://github.com/dadiaomengmeimei/claude-code-sourcemap-learning-notebook)

## License

MIT。被分析的 Claude Code 源码版权归 Anthropic 所有。本项目仅用于教育和研究目的。

---

**如果这个项目对你有帮助，请给一个 ⭐！关注 [@dadiaomengmeimei](https://github.com/dadiaomengmeimei) 获取更多 AI 源码分析。**
