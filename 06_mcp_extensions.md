# 第6章：MCP 协议、Skills 与扩展系统

## 学习目标
1. 理解 MCP (Model Context Protocol) 的核心概念与传输方式
2. 理解 MCP 工具包装、权限控制与 ToolSearch 延迟加载
3. 理解 Skills 系统的三层架构（bundled / disk-based / MCP）
4. 理解 bundled skill 的注册机制、文件提取与安全设计
5. 理解关键内置技能的 prompt 工程（simplify, skillify, remember, stuck）
6. 理解 Plugins 系统的信任分级

---

## 6.1 MCP 是什么？

**Model Context Protocol (MCP)** 是 Anthropic 提出的开放协议，
用于标准化 AI 模型与外部工具/数据源的交互。

```
┌──────────────┐  MCP Protocol  ┌──────────────┐
│  Claude Code  │ ◄──────────────────► │  MCP Server  │
│  (MCP Client) │   JSON-RPC 2.0      │  (任何语言)   │
└──────────────┘                      └──────────────┘
                                        可以提供：
                                        - Tools (工具)
                                        - Resources (资源)
                                        - Prompts (提示词)
```

### MCP 的三大能力

| 能力 | 说明 | 示例 |
|------|------|------|
| **Tools** | 可执行的操作 | 数据库查询、API 调用、文件操作 |
| **Resources** | 可读取的数据 | 文档、配置、实时数据 |
| **Prompts** | 预定义的提示词模板 | 代码审查模板、分析模板 |

### 传输方式

```typescript
// Claude Code 支持多种 MCP 传输方式
type McpTransport =
  | StdioClientTransport           // 标准输入/输出（最常用）
  | SSEClientTransport             // Server-Sent Events
  | StreamableHTTPClientTransport  // HTTP 流
  | WebSocketTransport             // WebSocket
  | SdkControlClientTransport     // SDK 控制通道
```

## 6.2 MCP 客户端实现

`services/mcp/client.ts` (116KB, 3349行) 是 MCP 客户端的核心实现。

### 连接管理

```typescript
export async function connectToServer(
  name: string,
  config: ScopedMcpServerConfig
): Promise<MCPServerConnection> {
  // 1. 根据配置创建传输层
  let transport: Transport;
  if (config.command) {
    // Stdio 模式：启动子进程
    transport = new StdioClientTransport({
      command: config.command,
      args: config.args,
      env: { ...subprocessEnv(), ...config.env },
    });
  } else if (config.url) {
    // HTTP/SSE/WebSocket 模式
    transport = createRemoteTransport(config.url, config);
  }

  // 2. 创建 MCP Client 并连接
  const client = new Client({ name: 'claude-code', version: '1.0.0' });
  await client.connect(transport);

  return { type: 'connected', name, client, cleanup: () => client.close() };
}
```

### 错误处理

```typescript
class McpAuthError extends Error {}        // OAuth token 过期 → 重新认证
class McpSessionExpiredError extends Error {} // 会话过期 → 重新连接
class McpToolCallError extends Error {}    // 工具调用失败 → 返回错误给 Claude
```

## 6.3 MCP 工具包装

MCP 服务器提供的工具会被包装为 Claude Code 的内置 Tool 格式：

```typescript
function createMcpTool(serverName, toolDef): Tool {
  return buildTool({
    // 名称格式: mcp__serverName__toolName
    name: `mcp__${normalizeNameForMCP(serverName)}__${normalizeNameForMCP(toolDef.name)}`,
    inputJSONSchema: toolDef.inputSchema,
    isMcp: true,
    mcpInfo: { serverName, toolName: toolDef.name },

    async call(input, context) {
      const result = await client.callTool({ name: toolDef.name, arguments: input });
      return processToolResult(result);
    },

    async checkPermissions() {
      return { behavior: 'passthrough', message: 'MCPTool requires permission.' };
    },
  });
}
```

## 6.4 ToolSearch — 延迟加载工具

当工具数量很多时（内置 + MCP），全部发送给 API 会消耗大量 token。
ToolSearch 机制允许延迟加载：

```
初始请求：
  tools: [
    Bash (完整 schema),
    Read (完整 schema),
    ToolSearch (完整 schema),          ← 搜索工具
    mcp__db__query (defer_loading),    ← 只有名称
    mcp__api__fetch (defer_loading),   ← 只有名称
  ]

当 Claude 需要使用延迟加载的工具时：
  1. Claude 调用 ToolSearch({ query: 'database query' })
  2. ToolSearch 返回匹配的工具列表
  3. 下次请求中包含完整的工具 schema
  4. Claude 可以正常调用该工具
```

某些 MCP 工具可以标记为 `alwaysLoad`（通过 `_meta['anthropic/alwaysLoad'] = true`），
始终包含完整 schema，不需要 ToolSearch。

---

## 6.5 Skills 系统 — 三层架构

Skills 是可复用的工作流，本质上是预定义的 prompt 模板。Claude Code 有三层 skill 来源：

```
┌─────────────────────────────────────────────────────────────────┐
│                    Skills 三层架构                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Layer 1: Bundled Skills (内置)                                  │
│  ├── 编译进 CLI 二进制文件                                       │
│  ├── 所有用户可用                                                │
│  ├── 注册在 src/skills/bundled/                                  │
│  └── 通过 registerBundledSkill() 注册                            │
│                                                                 │
│  Layer 2: Disk-based Skills (文件系统)                            │
│  ├── Markdown 文件 + frontmatter                                 │
│  ├── 三个搜索路径：                                              │
│  │   ├── ~/.claude/skills/        (用户级)                       │
│  │   ├── .claude/skills/          (项目级)                       │
│  │   └── managed/.claude/skills/  (企业策略级)                   │
│  └── 通过 loadSkillsDir() 加载                                   │
│                                                                 │
│  Layer 3: MCP Skills                                             │
│  ├── MCP 服务器提供的 prompt 模板                                │
│  ├── 通过 registerMCPSkillBuilders() 注册                        │
│  └── 动态发现，无需预安装                                        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 6.5.1 Disk-based Skill 的 Frontmatter 格式

```markdown
<!-- .claude/skills/review-pr/SKILL.md -->
---
name: review-pr
description: Review a pull request
allowed-tools:
  - Bash(gh:*)
  - Read
  - Grep
when_to_use: >
  Use when the user wants to review a PR.
  Examples: 'review this PR', 'check the changes'
argument-hint: "[PR number or URL]"
arguments:
  - pr_ref
context: fork
---

# PR Review Skill

Review PR `$pr_ref`:

1. Run `gh pr diff $pr_ref` to see changes
2. Check for code quality issues and security vulnerabilities
3. Provide a summary with recommendations

**Success criteria**: A structured review with actionable feedback.
```

### 6.5.2 Frontmatter 解析的完整字段

```typescript
export function parseSkillFrontmatterFields(
  frontmatter: FrontmatterData,
  markdownContent: string,
  resolvedName: string,
): {
  displayName: string | undefined
  description: string
  hasUserSpecifiedDescription: boolean
  allowedTools: string[]
  argumentHint: string | undefined
  argumentNames: string[]
  whenToUse: string | undefined
  version: string | undefined
  model: ReturnType<typeof parseUserSpecifiedModel> | undefined
  disableModelInvocation: boolean
  userInvocable: boolean
  hooks: HooksSettings | undefined
  context: 'inline' | 'fork' | undefined
  agent: string | undefined
  effort: EffortValue | undefined
  shell: FrontmatterShell | undefined
}
```

关键字段说明：

| 字段 | 作用 | 巧思 |
|------|------|------|
| `allowed-tools` | 限制 skill 可用的工具 | 支持 `Bash(gh:*)` 这样的参数化权限 |
| `when_to_use` | 告诉 Claude 何时自动调用 | 是 skill 自动发现的关键 |
| `context` | `inline` 或 `fork` | fork 在独立子 Agent 中运行，不污染主对话 |
| `agent` | 指定运行此 skill 的 Agent 类型 | 可以用自定义 Agent 执行 |
| `arguments` | 参数列表 | 支持 `$arg_name` 模板替换 |
| `shell` | 指定 shell 环境 | 用于 prompt 中的 shell 命令执行 |

---

## 6.6 Bundled Skills — 内置技能的注册机制

### 6.6.1 注册流程

```typescript
// bundledSkills.ts
export type BundledSkillDefinition = {
  name: string
  description: string
  aliases?: string[]
  whenToUse?: string
  allowedTools?: string[]
  model?: string
  disableModelInvocation?: boolean
  userInvocable?: boolean
  isEnabled?: () => boolean
  hooks?: HooksSettings
  context?: 'inline' | 'fork'
  agent?: string
  files?: Record<string, string>  // 附带的参考文件
  getPromptForCommand: (args: string, context: ToolUseContext) => Promise<ContentBlockParam[]>
}

export function registerBundledSkill(definition: BundledSkillDefinition): void {
  // 如果有附带文件，包装 getPromptForCommand 以支持懒提取
  if (files && Object.keys(files).length > 0) {
    skillRoot = getBundledSkillExtractDir(definition.name)
    let extractionPromise: Promise<string | null> | undefined
    const inner = definition.getPromptForCommand
    getPromptForCommand = async (args, ctx) => {
      // 懒提取：第一次调用时提取到磁盘，后续复用
      extractionPromise ??= extractBundledSkillFiles(definition.name, files)
      const extractedDir = await extractionPromise
      const blocks = await inner(args, ctx)
      if (extractedDir === null) return blocks
      return prependBaseDir(blocks, extractedDir)
    }
  }

  bundledSkills.push(command)
}
```

**巧思1：懒提取 + Promise 记忆化**

```typescript
// 记忆化的是 Promise（不是结果），所以并发调用者等待同一次提取
// 而不是各自发起独立的写入
extractionPromise ??= extractBundledSkillFiles(definition.name, files)
```

### 6.6.2 文件提取的安全设计

```typescript
// 安全写入：O_NOFOLLOW | O_EXCL 防止符号链接攻击
const SAFE_WRITE_FLAGS =
  process.platform === 'win32'
    ? 'wx'
    : fsConstants.O_WRONLY | fsConstants.O_CREAT | fsConstants.O_EXCL | O_NOFOLLOW

async function safeWriteFile(p: string, content: string): Promise<void> {
  const fh = await open(p, SAFE_WRITE_FLAGS, 0o600)  // owner-only
  try {
    await fh.writeFile(content, 'utf8')
  } finally {
    await fh.close()
  }
}

// 路径验证：防止目录遍历
function resolveSkillFilePath(baseDir: string, relPath: string): string {
  const normalized = normalize(relPath)
  if (
    isAbsolute(normalized) ||
    normalized.split(pathSep).includes('..') ||
    normalized.split('/').includes('..')
  ) {
    throw new Error(`bundled skill file path escapes skill dir: ${relPath}`)
  }
  return join(baseDir, normalized)
}
```

**巧思2：多层防御**

```
防御层1: getBundledSkillsRoot() 使用进程级 nonce → 路径不可预测
防御层2: mkdir 使用 0o700 → 只有 owner 可访问
防御层3: O_NOFOLLOW → 不跟随符号链接
防御层4: O_EXCL → 文件已存在则失败（不覆盖）
防御层5: resolveSkillFilePath → 拒绝 .. 和绝对路径
防御层6: 不 unlink+retry → 避免 unlink 跟随中间符号链接
```

### 6.6.3 启动时注册

```typescript
// bundled/index.ts
export function initBundledSkills(): void {
  // 无条件注册的技能
  registerUpdateConfigSkill()
  registerKeybindingsSkill()
  registerVerifySkill()
  registerDebugSkill()
  registerLoremIpsumSkill()
  registerSkillifySkill()
  registerRememberSkill()
  registerSimplifySkill()
  registerBatchSkill()
  registerStuckSkill()

  // Feature-gated 技能（按需加载）
  if (feature('KAIROS') || feature('KAIROS_DREAM')) {
    const { registerDreamSkill } = require('./dream.js')
    registerDreamSkill()
  }
  if (feature('AGENT_TRIGGERS')) {
    const { registerLoopSkill } = require('./loop.js')
    registerLoopSkill()
  }
  // ... 更多 feature-gated 技能
}
```

**巧思3：require() 而非 import**

Feature-gated 的技能使用 `require()` 而非 `import`。这是因为：
- `import` 是静态的，即使在 `if` 块中也会被 bundler 包含
- `require()` 是动态的，只有条件满足时才加载
- 这减少了外部构建的 bundle 大小

---

## 6.7 关键内置技能深度分析

### 6.7.1 /simplify — 三 Agent 并行代码审查

这是一个展示多 Agent 协作的典范技能：

```
/simplify 的执行流程：

Phase 1: 识别变更
  └── git diff (或 git diff HEAD)

Phase 2: 启动三个并行 Agent
  ├── Agent 1: Code Reuse Review
  │   └── 搜索现有工具函数，标记重复实现
  ├── Agent 2: Code Quality Review
  │   └── 检查冗余状态、参数膨胀、copy-paste、泄漏抽象
  └── Agent 3: Efficiency Review
      └── 检查不必要的工作、缺失并发、热路径膨胀

Phase 3: 汇总并修复
  └── 聚合三个 Agent 的发现，直接修复问题
```

注意 prompt 中的精心设计：
- "If a finding is a false positive, note it and move on — do not argue with the finding"
  → 防止 Claude 浪费 token 争论
- 每个 Agent 都收到完整的 diff → 避免上下文不足
- 三个 Agent 并行启动 → 利用 Coordinator 的并发能力

### 6.7.2 /skillify — 会话到技能的转化器

这是最复杂的内置技能，它能将当前会话的工作流程转化为可复用的 skill：

```typescript
const SKILLIFY_PROMPT = `# Skillify {{userDescriptionBlock}}

You are capturing this session's repeatable process as a reusable skill.

## Your Session Context

Here is the session memory summary:
<session_memory>
{{sessionMemory}}
</session_memory>

Here are the user's messages during this session:
<user_messages>
{{userMessages}}
</user_messages>
`
```

**巧思4：动态上下文注入**

```typescript
async getPromptForCommand(args, context) {
  // 获取会话记忆（Claude 对当前会话的理解）
  const sessionMemory = (await getSessionMemoryContent()) ?? 'No session memory available.'

  // 提取用户消息（只取 compact 边界之后的，避免过长）
  const userMessages = extractUserMessages(
    getMessagesAfterCompactBoundary(context.messages),
  )

  // 模板替换
  const prompt = SKILLIFY_PROMPT
    .replace('{{sessionMemory}}', sessionMemory)
    .replace('{{userMessages}}', userMessages.join('\n\n---\n\n'))
    .replace('{{userDescriptionBlock}}', userDescriptionBlock)

  return [{ type: 'text', text: prompt }]
}
```

/skillify 的 prompt 设计了 4 轮交互式访谈：
1. **Round 1**: 确认名称、描述、目标
2. **Round 2**: 确认步骤、参数、inline/fork、保存位置
3. **Round 3**: 逐步深入每个步骤的细节
4. **Round 4**: 确认触发条件和注意事项

### 6.7.3 /remember — 记忆层级管理

```
/remember 的记忆层级：

┌─────────────────────────────────────────────┐
│  CLAUDE.md          (项目级，所有贡献者共享)  │
│  ├── 项目约定：use bun not npm              │
│  ├── 代码风格：prefer functional style      │
│  └── 测试命令：bun test                     │
├─────────────────────────────────────────────┤
│  CLAUDE.local.md    (个人级，不提交到 git)    │
│  ├── 个人偏好：I prefer concise responses   │
│  └── 工作流：don't auto-commit              │
├─────────────────────────────────────────────┤
│  Auto-memory        (自动提取的工作笔记)      │
│  ├── 会话中观察到的模式                      │
│  └── 临时上下文                              │
├─────────────────────────────────────────────┤
│  Team memory        (组织级，跨仓库)          │
│  └── 部署流程、staging 地址等                │
└─────────────────────────────────────────────┘
```

/remember 的核心工作是**分类和提升**：
- 分析 auto-memory 中的每条记录
- 判断它应该提升到哪个层级
- 检测跨层级的重复和冲突
- 生成结构化报告，等待用户确认后再修改

### 6.7.4 /stuck — 诊断卡住的会话（内部工具）

```typescript
export function registerStuckSkill(): void {
  // 只对 Anthropic 内部用户可用
  if (process.env.USER_TYPE !== 'ant') return

  registerBundledSkill({
    name: 'stuck',
    description: '[ANT-ONLY] Investigate frozen/stuck/slow Claude Code sessions...',
    // ...
  })
}
```

/stuck 的 prompt 是一个完整的诊断手册：
- 列出所有 Claude Code 进程（`ps -axo pid=,pcpu=,rss=,etime=,state=,comm=,command=`）
- 检查 CPU、内存、进程状态（D/T/Z）
- 检查子进程（`pgrep -lP <pid>`）
- 可选：macOS 上的 `sample <pid> 3` 获取原生栈采样
- 如果发现问题，自动发送到 Slack #claude-code-feedback

**巧思5：两条消息结构**

```
Slack 消息结构：
  1. 顶层消息：一行简短摘要（hostname + 版本 + 症状）
  2. 线程回复：完整诊断数据

原因：保持频道可扫描，详细信息在线程中
```

---

## 6.8 Skill 加载与去重

### 6.8.1 文件系统 Skill 加载

```typescript
// loadSkillsDir.ts — 从多个目录加载 skills
// 搜索路径优先级：
//   1. policySettings (企业策略)
//   2. userSettings (用户级)
//   3. projectSettings (项目级)
//   4. plugin (插件提供)

// 去重策略：使用 realpath 解析符号链接
async function getFileIdentity(filePath: string): Promise<string | null> {
  try {
    return await realpath(filePath)
  } catch {
    return null
  }
}
```

**巧思6：为什么用 realpath 而不是 inode？**

```
// 注释原文：
// Uses realpath to resolve symlinks, which is filesystem-agnostic and avoids
// issues with filesystems that report unreliable inode values (e.g., inode 0 on
// some virtual/container/NFS filesystems, or precision loss on ExFAT).
```

某些文件系统（NFS、容器、ExFAT）的 inode 值不可靠（可能返回 0），
导致不同文件被误判为相同。realpath 通过路径规范化来去重，更可靠。

### 6.8.2 Skill Token 估算

```typescript
// 只估算 frontmatter 的 token（name + description + whenToUse）
// 不估算完整内容（内容只在调用时加载）
export function estimateSkillFrontmatterTokens(skill: Command): number {
  const frontmatterText = [skill.name, skill.description, skill.whenToUse]
    .filter(Boolean)
    .join(' ')
  return roughTokenCountEstimation(frontmatterText)
}
```

这是一个重要的优化：skill 的完整 prompt 可能很长（/simplify 有 ~70 行），
但只有 frontmatter 信息需要在每次 API 请求中发送（用于 Claude 决定是否调用）。

---

## 6.9 Plugins 系统

Plugins 是更强大的扩展机制，可以提供工具、技能、Agent 等：

```typescript
type Plugin = {
  name: string
  version: string
  tools?: Tool[]              // 自定义工具
  skills?: Skill[]            // 自定义技能
  agents?: AgentDefinition[]  // 自定义 Agent
  commands?: Command[]        // 自定义 /slash 命令
  hooks?: HookDefinition[]    // 自定义 hooks
}
```

### 插件信任分级

```typescript
type PluginSource =
  | 'builtin'         // 内置（完全信任）
  | 'plugin'          // 已安装的插件（管理员信任）
  | 'policySettings'  // 企业策略（管理员信任）
  | 'user'            // 用户自定义（受限信任）

// strictPluginOnlyCustomization 模式下：
// - 只有 builtin/plugin/policySettings 来源的扩展可以加载
// - 用户自定义的 MCP、hooks、agents 被阻止
```

---

## 6.10 MCP 配置层级

```json
// ~/.claude/settings.json (用户级)
{
  "mcpServers": {
    "my-database": {
      "command": "node",
      "args": ["db-server.js"],
      "env": { "DB_URL": "postgres://..." }
    }
  }
}

// .claude/settings.json (项目级)
{
  "mcpServers": {
    "project-tools": {
      "command": "python",
      "args": ["tools/mcp_server.py"]
    }
  }
}
```

合并规则：项目级覆盖用户级（同名时），企业策略优先级最高。

---

## 本章小结

| 概念 | 关键点 |
|------|--------|
| MCP 协议 | 标准化的 AI-工具交互协议，支持 Tools/Resources/Prompts |
| ToolSearch | 延迟加载机制，减少初始 token 消耗 |
| Skills 三层 | bundled（编译进二进制）→ disk-based（Markdown）→ MCP（动态） |
| Bundled Skill | registerBundledSkill() 注册，懒提取文件，Promise 记忆化 |
| 文件安全 | O_NOFOLLOW + O_EXCL + nonce 目录 + 路径验证，6 层防御 |
| /simplify | 三 Agent 并行审查（复用、质量、效率） |
| /skillify | 4 轮交互式访谈，将会话转化为可复用 skill |
| /remember | 记忆层级管理，分类 + 提升 + 去重 |
| Plugins | 完整的扩展包，信任分级（builtin > plugin > user） |

### 下一章预告
第7章将深入 **Prompt 工程**——理解 Claude Code 的 system prompt 设计哲学。
