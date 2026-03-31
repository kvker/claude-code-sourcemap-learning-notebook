
# Claude Code Sourcemap Learning Notebook

[![GitHub](https://img.shields.io/badge/GitHub-dadiaomengmeimei-blue?logo=github)](https://github.com/dadiaomengmeimei/claude-code-sourcemap-learning-notebook)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)

> 深入解析 Anthropic Claude Code CLI 工具的内部架构，从入口到工具系统、权限安全、查询循环、多 Agent 协作、MCP 扩展协议、Voice 模式与 Buddy 系统——逐层拆解 50 万行 TypeScript 代码。

## 项目简介

[Claude Code](https://docs.anthropic.com/en/docs/claude-code) 是 Anthropic 开发的 AI 编程助手 CLI 工具，允许开发者从终端直接与 Claude AI 交互，执行代码编辑、文件搜索、Shell 命令、代码审查等软件工程任务。

本仓库是一套**结构化的学习笔记**，以 Markdown 文档的形式，对 Claude Code 的核心源码进行深入浅出的解析。适合以下读者：

- **AI 工具研究者** — 想了解工业级 AI Agent 系统的架构设计
- **安全研究者** — 想研究 AI 工具的权限控制和安全机制
- **架构师** — 想学习大规模 TypeScript 项目的工程实践
- **Agent 开发者** — 想了解多 Agent 协作和工具系统的设计模式

## Claude Code 技术概况

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

## 学习路线

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
           │ 05 多Agent   │ │ 06 MCP/Skills  │
           │ +Coordinator │ │ (扩展生态)     │
           └──────┬───────┘ └───────┬────────┘
                  │                 │
                  └────────┬────────┘
                           │
                  ┌────────▼────────┐
                  │ 07 Prompt       │
                  │ (AI的灵魂)      │
                  └────────┬────────┘
                           │
                  ┌────────▼────────┐
                  │ 08 Voice+Buddy  │
                  │ (特色功能)      │
                  └─────────────────┘
```

### 章节详情

| # | 文件 | 主题 | 核心源码文件 | 预计时间 |
|---|------|------|-------------|----------|
| 00 | [00_index.md](00_index.md) | 总目录与架构全景图 | — | 10min |
| 01 | [01_architecture_overview.md](01_architecture_overview.md) | 全局架构鸟瞰 | `main.tsx`, `App.tsx`, `QueryEngine.ts` | 30min |
| 02 | [02_tool_system.md](02_tool_system.md) | 工具系统深度解析 | `Tool.ts`, `tools.ts`, `GlobTool.ts` | 45min |
| 03 | [03_permission_security.md](03_permission_security.md) | 权限与安全系统 | `permissions.ts`, `filesystem.ts` | 45min |
| 04 | [04_query_loop_api.md](04_query_loop_api.md) | 查询循环与 API 交互 (深度版) | `query.ts`, `StreamingToolExecutor.ts` | 50min |
| 05 | [05_multi_agent_system.md](05_multi_agent_system.md) | 多 Agent 系统 + Coordinator (深度版) | `AgentTool.tsx`, `runAgent.ts`, `coordinatorMode.ts` | 50min |
| 06 | [06_mcp_extensions.md](06_mcp_extensions.md) | MCP、Skills 与扩展系统 | `mcp/client.ts`, `skills/`, `bundledSkills.ts` | 45min |
| 07 | [07_prompt_engineering.md](07_prompt_engineering.md) | Prompt Engineering 深度解析 | `prompts.ts`, `BashTool/prompt.ts`, `compact/prompt.ts` | 60min |
| 08 | [08_voice_buddy.md](08_voice_buddy.md) | Voice 模式与 Buddy 系统 | `voiceStreamSTT.ts`, `useVoice.ts`, `buddy/` | 30min |

**总计约 5.5 小时**，可以系统性地理解 Claude Code 的核心架构。

## 推荐学习路径

### 快速入门 (2 小时)
```
00 → 01 → 02 → 04
```
理解：入口文件 → 工具定义 → 查询循环

### 安全研究 (1.5 小时)
```
00 → 03 → 02(权限部分)
```
理解：权限模型 → 命令安全分析 → 沙箱机制

### Agent 研究 (2.5 小时)
```
00 → 05 → 04 → 06
```
理解：Agent 架构 + Coordinator → 查询循环 → MCP/Skills 扩展

### Prompt 研究 (1 小时)
```
00 → 07
```
理解：系统提示词 → 工具提示词 → Agent 提示词 → 压缩提示词 → 缓存优化

### 特色功能 (30 分钟)
```
08
```
理解：Voice hold-to-talk → WebSocket STT → Buddy 确定性生成 → ASCII 动画

### 完整学习 (5.5 小时)
```
00 → 01 → 02 → 03 → 04 → 05 → 06 → 07 → 08
```

## 架构全景图

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

## 关键源码文件速查

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

## 核心设计理念

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

## 每章核心发现

| 章节 | 最有价值的发现 |
|------|---------------|
| 01 架构 | React/Ink 终端 UI + 函数式状态管理，CLI 也能用 React |
| 02 工具 | `buildTool()` 工厂模式 + Zod schema 即 prompt，18+ Feature Flags |
| 03 安全 | 5 层纵深防御，bypass 模式也有安全底线（.git/.claude 不可绕过） |
| 04 查询 | `StreamingToolExecutor` 流式并发执行，三层 AbortController 级联取消，5 层压缩管线 |
| 05 Agent | 默认隔离显式共享，Fork prompt cache 共享，Coordinator 编排者模式，10 步 finally 清理 |
| 06 MCP/Skills | Skills 三层架构，bundled skill 懒提取 + 6 层安全防御，/simplify 三 Agent 并行审查 |
| 07 Prompt | 7大类40+个prompt文件，静态/动态分离缓存优化，Meta-Prompt生成Agent |
| 08 Voice/Buddy | 录音先行连接后补消除延迟，Buddy 确定性生成 + Bones 不持久化防伪造 |

## 如何使用

### 方式一：GitHub 在线浏览
GitHub 原生支持 Markdown 渲染，直接点击 `.md` 文件即可阅读。

### 方式二：VS Code
直接用 VS Code 打开 `.md` 文件，使用 Markdown Preview 阅读。

### 方式三：任何 Markdown 阅读器
所有章节都是标准 Markdown 格式，可以用任何支持 Markdown 的工具阅读。

## 免责声明

本项目仅用于**教育和研究目的**。源码来自通过 npm 包 source map 文件公开暴露的 Claude Code TypeScript 源码。本项目不包含任何原始源码文件，仅包含对架构和设计模式的分析笔记。

## License

本学习笔记以 MIT License 发布。请注意，被分析的 Claude Code 源码版权归 Anthropic 所有。

## 贡献

欢迎提交 Issue 和 PR！如果你发现了有趣的设计模式或安全机制，欢迎补充到笔记中。

**贡献方式：**
1. Fork 本仓库
2. 创建你的分支 (`git checkout -b feature/amazing-discovery`)
3. 提交修改 (`git commit -m 'Add: 发现了 XXX 的有趣设计'`)
4. 推送到分支 (`git push origin feature/amazing-discovery`)
5. 提交 Pull Request

## 相关链接

- [Claude Code 官方文档](https://docs.anthropic.com/en/docs/claude-code)
- [Anthropic 官网](https://www.anthropic.com/)
- [MCP 协议规范](https://modelcontextprotocol.io/)
- [本项目 GitHub](https://github.com/dadiaomengmeimei/claude-code-sourcemap-learning-notebook)

---

**如果这个项目对你有帮助，请给一个 Star！**

**关注作者 [@dadiaomengmeimei](https://github.com/dadiaomengmeimei) 获取更多 AI 工具源码分析！**
