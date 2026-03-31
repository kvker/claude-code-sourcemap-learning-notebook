# 第8章：Voice 模式与 Buddy 系统

> 本章覆盖 Claude Code 的两个特色功能：语音输入系统和虚拟伴侣系统。
> 它们展示了 CLI 工具如何在保持专业性的同时加入创意功能。

## 学习目标
1. 理解 Voice 模式的完整技术栈（hold-to-talk → 录音 → WebSocket STT → 注入）
2. 理解 Voice 的多层可用性检查（feature gate + OAuth + GrowthBook kill-switch）
3. 理解 Buddy 系统的确定性生成机制（hash → PRNG → bones）
4. 理解 Buddy 的 ASCII sprite 动画系统
5. 理解 Buddy 的 prompt 注入与 Claude 的协作方式

---

## 8.1 Voice 模式 — Hold-to-Talk 语音输入

### 8.1.1 技术栈概览

```
用户按住快捷键
  │
  ▼
┌──────────────────┐
│ useVoice.ts      │  React Hook，管理录音状态机
│ (hold-to-talk)   │
└────────┬─────────┘
         │ 音频数据 (PCM 16-bit, 16kHz, mono)
         ▼
┌──────────────────┐     ┌──────────────────────┐
│ 录音后端          │     │ voiceStreamSTT.ts     │
│ ├── native module │────▶│ WebSocket 客户端       │
│ │   (macOS)       │     │ → Anthropic voice_stream│
│ └── SoX fallback  │     │ → Deepgram Nova 3 STT  │
└──────────────────┘     └──────────┬───────────┘
                                    │ TranscriptText / TranscriptEndpoint
                                    ▼
                          ┌──────────────────┐
                          │ 文本注入到输入框   │
                          │ onTranscript()    │
                          └──────────────────┘
```

### 8.1.2 三层可用性检查

```typescript
// voiceModeEnabled.ts

// 层1: Feature gate（编译时）
// 只有 ant 构建包含 VOICE_MODE feature
function isVoiceGrowthBookEnabled(): boolean {
  return feature('VOICE_MODE')
    ? !getFeatureValue_CACHED_MAY_BE_STALE('tengu_amber_quartz_disabled', false)
    : false
}

// 层2: OAuth 认证检查
// Voice 使用 claude.ai 的 voice_stream 端点，需要 OAuth token
function hasVoiceAuth(): boolean {
  if (!isAnthropicAuthEnabled()) return false
  const tokens = getClaudeAIOAuthTokens()
  return Boolean(tokens?.accessToken)
}

// 层3: 完整运行时检查
export function isVoiceModeEnabled(): boolean {
  return hasVoiceAuth() && isVoiceGrowthBookEnabled()
}
```

**巧思1：GrowthBook kill-switch 的默认值设计**

```typescript
// 注释原文：
// Default `false` means a missing/stale disk cache reads as "not killed"
// — so fresh installs get voice working immediately without waiting for
// GrowthBook init.
!getFeatureValue_CACHED_MAY_BE_STALE('tengu_amber_quartz_disabled', false)
```

kill-switch 的默认值是 `false`（未禁用），这意味着：
- 新安装的用户不需要等 GrowthBook 初始化就能用语音
- 只有当 Anthropic 主动推送 `true` 时才会禁用
- 这是"默认开启，紧急关闭"的模式

### 8.1.3 WebSocket 连接与音频传输

```typescript
// voiceStreamSTT.ts
export async function connectVoiceStream(
  callbacks: VoiceStreamCallbacks,
  options?: { language?: string; keyterms?: string[] },
): Promise<VoiceStreamConnection | null> {
  // 刷新 OAuth token
  await checkAndRefreshOAuthTokenIfNeeded()

  // 连接到 api.anthropic.com（不是 claude.ai）
  // 原因：claude.ai 的 Cloudflare 使用 TLS 指纹检测，
  // 会 challenge 非浏览器客户端
  const wsBaseUrl = getOauthConfig()
    .BASE_API_URL.replace('https://', 'wss://')
    .replace('http://', 'ws://')

  const params = new URLSearchParams({
    encoding: 'linear16',
    sample_rate: '16000',
    channels: '1',
    endpointing_ms: '300',    // 300ms 静音 → 端点检测
    utterance_end_ms: '1000', // 1s 静音 → 语句结束
    language: options?.language ?? 'en',
  })

  // Deepgram Nova 3 路由（通过 conversation-engine）
  if (isNova3GateEnabled) {
    params.set('use_conversation_engine', 'true')
    params.set('stt_provider', 'deepgram-nova3')
  }

  // Keyterms boosting（提高特定术语的识别准确率）
  if (options?.keyterms?.length) {
    for (const term of options.keyterms) {
      params.append('keyterms', term)
    }
  }

  const ws = new WebSocket(url, wsOptions)
  // ...
}
```

**巧思2：为什么用 api.anthropic.com 而不是 claude.ai？**

```
claude.ai 的 Cloudflare zone 使用 TLS 指纹检测（JA3 fingerprint）。
CLI 的 WebSocket 客户端不是浏览器，会被 challenge。

api.anthropic.com 是同一个 private-api pod，同样的 OAuth Bearer auth，
但 CF zone 不阻止非浏览器客户端。

Desktop 版本仍然用 claude.ai（Swift URLSession 有浏览器级的 JA3 指纹）。
```

### 8.1.4 录音与缓冲策略

```typescript
// useVoice.ts — startRecordingSession

// 关键设计：录音立即开始，不等 WebSocket 连接
// 音频先缓冲，WebSocket 就绪后一次性 flush
const audioBuffer: Buffer[] = []

// 立即开始录音
const started = await voiceModule.startRecording(
  (chunk: Buffer) => {
    const owned = Buffer.from(chunk)  // 复制！native module 可能复用 buffer
    if (connectionRef.current) {
      connectionRef.current.send(owned)  // WebSocket 已连接 → 直接发送
    } else {
      audioBuffer.push(owned)  // 未连接 → 缓冲
    }
  },
)

// WebSocket 连接就绪后 flush 缓冲
onReady: conn => {
  // 合并为 ~1s 的切片，减少 WS 帧数
  const SLICE_TARGET_BYTES = 32_000  // 32KB ≈ 1s of 16kHz 16-bit mono
  const slices: Buffer[][] = [[]]
  for (const chunk of audioBuffer) {
    if (sliceBytes + chunk.length > SLICE_TARGET_BYTES) {
      slices.push([])
      sliceBytes = 0
    }
    slices[slices.length - 1]!.push(chunk)
    sliceBytes += chunk.length
  }
  for (const slice of slices) {
    conn.send(Buffer.concat(slice))
  }
  audioBuffer.length = 0
}
```

**巧思3：录音先行，连接后补**

```
传统方式：
  按键 → 连接 WebSocket (1-2s) → 开始录音
  用户说的前 1-2 秒被丢失

Claude Code 的方式：
  按键 → 立即录音 + 同时连接 WebSocket
  WebSocket 就绪 → flush 缓冲的音频
  用户说的每一个字都被捕获
```

### 8.1.5 Finalize 与静默丢弃检测

```typescript
// 当用户松开按键时
finalize(): Promise<FinalizeSource> {
  return new Promise<FinalizeSource>(resolve => {
    // 安全超时：5s（WebSocket 挂起的最后防线）
    const safetyTimer = setTimeout(
      () => resolveFinalize?.('safety_timeout'),
      5_000,
    )

    // 无数据超时：1.5s（服务器没有返回任何 transcript）
    const noDataTimer = setTimeout(
      () => resolveFinalize?.('no_data_timeout'),
      1_500,
    )

    // 延迟发送 CloseStream（等事件循环清空音频回调）
    setTimeout(() => {
      ws.send('{"type":"CloseStream"}')
    }, 0)
  })
}
```

**巧思4：no_data_timeout 检测静默丢弃**

```
~1% 的会话会遇到"粘性坏 pod"：
  - CE (conversation-engine) pod 接受音频但不返回 transcript
  - 这是 anthropics/anthropic#287008 的 session-sticky 变体

检测方式：
  finalize() 通过 no_data_timeout 解析
  + hadAudioSignal = true（确实有声音）
  → 判定为静默丢弃

恢复方式：
  重放完整音频到新的 WebSocket 连接（一次重试机会）
  fullAudioRef 保存了所有音频块（~32KB/s × 60s ≈ 2MB）
```

### 8.1.6 Auto-repeat 按键检测

```
Hold-to-talk 的挑战：终端无法直接检测"按键释放"。

解决方案：利用操作系统的按键重复机制
  1. 用户按住快捷键
  2. OS 在 ~500ms 延迟后开始发送重复按键事件
  3. useVoice 检测到重复事件 → 设置 seenRepeat = true
  4. 启动 RELEASE_TIMEOUT_MS 定时器
  5. 如果定时器到期前没有新的按键事件 → 判定为松开
  6. 停止录音，发送 finalize

关键：定时器只在检测到重复后才启动，
避免在 OS 按键重复延迟期间误判为松开。
```

---

## 8.2 Buddy 系统 — 虚拟伴侣

### 8.2.1 系统概览

Buddy 是 Claude Code 的虚拟宠物/伴侣系统。用户可以通过 `/buddy` 命令"孵化"
一个伴侣，它会在输入框旁边显示 ASCII sprite 动画，偶尔在对话气泡中发表评论。

```
┌─────────────────────────────────────────────────────────┐
│  Buddy 系统架构                                          │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  companion.ts    → 确定性生成（hash → PRNG → bones）    │
│  types.ts        → 类型定义（species, rarity, stats）   │
│  sprites.ts      → ASCII 动画帧（18 种生物 × 3 帧）    │
│  prompt.ts       → Prompt 注入（告诉 Claude 有伴侣）   │
│  CompanionSprite → React 组件（渲染 + 动画循环）        │
│  useBuddyNotification → 彩虹 /buddy 提示               │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

### 8.2.2 确定性生成 — hash(userId) → 伴侣

每个用户的伴侣是**确定性**的 — 相同的 userId 永远生成相同的伴侣：

```typescript
// companion.ts
const SALT = 'friend-2026-401'

export function roll(userId: string): Roll {
  const key = userId + SALT
  // 缓存：同一个 userId 只计算一次
  if (rollCache?.key === key) return rollCache.value
  const value = rollFrom(mulberry32(hashString(key)))
  rollCache = { key, value }
  return value
}
```

**巧思5：Mulberry32 PRNG**

```typescript
// 一个极简的 32-bit 种子 PRNG，足够用于选鸭子
function mulberry32(seed: number): () => number {
  let a = seed >>> 0
  return function () {
    a |= 0
    a = (a + 0x6d2b79f5) | 0
    let t = Math.imul(a ^ (a >>> 15), 1 | a)
    t = (t + Math.imul(t ^ (t >>> 7), 61 | t)) ^ t
    return ((t ^ (t >>> 14)) >>> 0) / 4294967296
  }
}
```

注释说得很直白："tiny seeded PRNG, good enough for picking ducks"。
不需要密码学安全的随机数，只需要确定性和均匀分布。

### 8.2.3 Bones vs Soul — 分离的持久化策略

```typescript
// Bones（骨架）— 确定性部分，从 hash(userId) 生成，不持久化
type CompanionBones = {
  rarity: Rarity       // common(60%) / uncommon(25%) / rare(10%) / epic(4%) / legendary(1%)
  species: Species     // 18 种生物
  eye: Eye             // 6 种眼睛样式
  hat: Hat             // 8 种帽子（common 没有帽子）
  shiny: boolean       // 1% 概率闪光
  stats: Record<StatName, number>  // DEBUGGING, PATIENCE, CHAOS, WISDOM, SNARK
}

// Soul（灵魂）— 模型生成部分，持久化到 config
type CompanionSoul = {
  name: string         // 模型取的名字
  personality: string  // 模型生成的性格描述
}

// 存储的只有 Soul + hatchedAt
type StoredCompanion = CompanionSoul & { hatchedAt: number }
```

**巧思6：为什么 Bones 不持久化？**

```
// 注释原文：
// Bones never persist so species renames and SPECIES-array edits can't
// break stored companions, and editing config.companion can't fake a rarity.

好处：
1. 如果开发者重命名了某个 species → 不会破坏已有伴侣
2. 如果调整了 SPECIES 数组 → 不会导致反序列化错误
3. 用户不能通过编辑 config 文件伪造 legendary 稀有度
4. 每次读取时重新生成 → 始终与最新代码一致
```

### 8.2.4 Species 名称的编码技巧

```typescript
// types.ts — 为什么 species 名称用 charCode 编码？
const c = String.fromCharCode
export const duck = c(0x64,0x75,0x63,0x6b) as 'duck'
export const goose = c(0x67,0x6f,0x6f,0x73,0x65) as 'goose'
// ...
```

**巧思7：绕过构建检查**

```
// 注释原文：
// One species name collides with a model-codename canary in excluded-strings.txt.
// The check greps build output (not source), so runtime-constructing the value
// keeps the literal out of the bundle while the check stays armed for the
// actual codename.

Anthropic 的构建系统会检查输出中是否包含模型代号（防止泄露）。
某个 species 名称恰好和一个模型代号冲突。
用 charCode 构造字符串 → 构建输出中没有字面量 → 检查通过。
所有 species 统一使用这种编码，保持一致性。
```

### 8.2.5 ASCII Sprite 动画系统

每个 species 有 3 帧动画，每帧 5 行 × 12 字符宽：

```
帧 0 (idle):          帧 1 (fidget):        帧 2 (special):
    __                    __                    __
  <(· )___              <(· )___              <(· )___
   (  ._>                (  ._>                (  .__>
    `--´                  `--´~                 `--´
```

```typescript
// sprites.ts
export function renderSprite(bones: CompanionBones, frame = 0): string[] {
  const frames = BODIES[bones.species]
  const body = frames[frame % frames.length]!.map(line =>
    line.replaceAll('{E}', bones.eye),  // 替换眼睛占位符
  )
  const lines = [...body]

  // 帽子只在第 0 行为空时显示（某些帧用第 0 行做烟雾/天线效果）
  if (bones.hat !== 'none' && !lines[0]!.trim()) {
    lines[0] = HAT_LINES[bones.hat]
  }

  // 如果所有帧的第 0 行都是空的，且没有帽子 → 删除空行节省空间
  if (!lines[0]!.trim() && frames.every(f => !f[0]!.trim())) lines.shift()

  return lines
}
```

**巧思8：帽子与动画帧的冲突处理**

```
帧 0: 空行 → 可以放帽子
帧 1: 空行 → 可以放帽子
帧 2: "   ~    ~   " → 烟雾效果，不能放帽子

解决方案：只在 lines[0].trim() 为空时替换帽子。
这样 dragon 的帧 2 可以显示烟雾而不是帽子。
```

### 8.2.6 Stats 生成 — 一个 peak，一个 dump

```typescript
function rollStats(rng: () => number, rarity: Rarity): Record<StatName, number> {
  const floor = RARITY_FLOOR[rarity]  // common:5, uncommon:15, rare:25, epic:35, legendary:50
  const peak = pick(rng, STAT_NAMES)
  let dump = pick(rng, STAT_NAMES)
  while (dump === peak) dump = pick(rng, STAT_NAMES)  // 确保不同

  const stats = {} as Record<StatName, number>
  for (const name of STAT_NAMES) {
    if (name === peak) {
      stats[name] = Math.min(100, floor + 50 + Math.floor(rng() * 30))
    } else if (name === dump) {
      stats[name] = Math.max(1, floor - 10 + Math.floor(rng() * 15))
    } else {
      stats[name] = floor + Math.floor(rng() * 40)
    }
  }
  return stats
}
```

这创造了有个性的伴侣：每个都有一个突出的强项和一个明显的弱项，
而不是所有属性都平庸。稀有度越高，基础值越高。

### 8.2.7 Prompt 注入 — Claude 如何与 Buddy 共存

```typescript
// prompt.ts
export function companionIntroText(name: string, species: string): string {
  return `# Companion

A small ${species} named ${name} sits beside the user's input box and
occasionally comments in a speech bubble. You're not ${name} — it's a
separate watcher.

When the user addresses ${name} directly (by name), its bubble will answer.
Your job in that moment is to stay out of the way: respond in ONE line or
less, or just answer any part of the message meant for you. Don't explain
that you're not ${name} — they know. Don't narrate what ${name} might say
— the bubble handles that.`
}
```

**巧思9：角色分离的 prompt 设计**

```
关键指令：
1. "You're not {name}" → 明确 Claude 和 Buddy 是不同实体
2. "stay out of the way" → 用户和 Buddy 对话时 Claude 退让
3. "Don't explain that you're not {name}" → 防止 Claude 打破沉浸感
4. "Don't narrate what {name} might say" → 防止 Claude 代替 Buddy 说话

这是一个精心设计的"共存协议"：
  Claude 处理技术工作
  Buddy 处理情感互动
  两者不互相干扰
```

### 8.2.8 Teaser 窗口 — 时间限定的发现机制

```typescript
// useBuddyNotification.tsx

// 本地时间（不是 UTC）— 24 小时滚动波跨时区
// 持续的 Twitter 热度，而不是单一的 UTC 午夜高峰
// Teaser 窗口：仅 2026 年 4 月 1-7 日。命令之后永久可用。
export function isBuddyTeaserWindow(): boolean {
  if ("external" === 'ant') return true  // Anthropic 内部始终可用
  const d = new Date()
  return d.getFullYear() === 2026 && d.getMonth() === 3 && d.getDate() <= 7
}

export function isBuddyLive(): boolean {
  if ("external" === 'ant') return true
  const d = new Date()
  return d.getFullYear() > 2026 || (d.getFullYear() === 2026 && d.getMonth() >= 3)
}
```

**巧思10：本地时间而非 UTC**

```
// 注释原文：
// Local date, not UTC — 24h rolling wave across timezones. Sustained Twitter
// buzz instead of a single UTC-midnight spike, gentler on soul-gen load.

如果用 UTC：
  UTC 午夜 → 全球同时发现 → Twitter 单一高峰 → soul-gen 服务器过载

用本地时间：
  每个时区的午夜 → 24 小时内逐渐扩散 → 持续的社交媒体热度 → 平滑的服务器负载
```

启动时，如果用户还没有伴侣且在 teaser 窗口内，会显示彩虹色的 `/buddy` 提示：

```typescript
addNotification({
  key: 'buddy-teaser',
  jsx: <RainbowText text="/buddy" />,
  priority: 'immediate',
  timeoutMs: 15_000,  // 15 秒后消失
})
```

### 8.2.9 缓存策略

```typescript
// companion.ts
// 从三个热路径调用（500ms sprite tick, 每次按键的 PromptInput, 每轮 observer）
// 使用相同的 userId → 缓存确定性结果
let rollCache: { key: string; value: Roll } | undefined
export function roll(userId: string): Roll {
  const key = userId + SALT
  if (rollCache?.key === key) return rollCache.value
  const value = rollFrom(mulberry32(hashString(key)))
  rollCache = { key, value }
  return value
}
```

单条目缓存足够了，因为同一个进程中 userId 不会变。

---

## 本章小结

### Voice 模式

| # | 巧思 | 解决的问题 |
|---|------|------------|
| 1 | kill-switch 默认 false | 新安装不等 GrowthBook 就能用 |
| 2 | api.anthropic.com | 绕过 claude.ai 的 TLS 指纹检测 |
| 3 | 录音先行，连接后补 | 消除 1-2s 的 WebSocket 连接延迟 |
| 4 | no_data_timeout | 检测并恢复静默丢弃（~1% 会话） |

### Buddy 系统

| # | 巧思 | 解决的问题 |
|---|------|------------|
| 5 | Mulberry32 PRNG | 确定性生成，"good enough for picking ducks" |
| 6 | Bones 不持久化 | 防止 species 重命名破坏存档，防止伪造稀有度 |
| 7 | charCode 编码 species | 绕过构建系统的模型代号检查 |
| 8 | 帽子与动画帧冲突处理 | 烟雾/天线帧不被帽子覆盖 |
| 9 | 角色分离 prompt | Claude 和 Buddy 各司其职，不互相干扰 |
| 10 | 本地时间 teaser | 24h 滚动波，平滑服务器负载和社交媒体热度 |
