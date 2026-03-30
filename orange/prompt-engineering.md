# OpenClaw 提示词工程（Prompt Engineering）分析文档

## 1. 概述

OpenClaw 的提示词工程是一个高度模块化、动态生成的系统。它通过多个独立的组件构建完整的 System Prompt，将用户配置、运行时信息、技能、内存等上下文注入到 AI 代理的上下文中。

---

## 2. 核心文件位置

### 2.1 主要文件

| 文件路径 | 描述 |
|---------|------|
| `src/agents/system-prompt.ts` | 核心 System Prompt 构建器 |
| `src/agents/system-prompt-params.ts` | System Prompt 参数构建 |
| `src/agents/system-prompt-report.ts` | System Prompt 报告生成 |
| `src/agents/pi-embedded-runner/system-prompt.ts` | 嵌入式运行时的 System Prompt |
| `src/agents/workspace.ts` | Workspace bootstrap 文件定义 |
| `src/agents/skills/workspace.ts` | Skills 提示词构建 |
| `src/agents/bootstrap-files.ts` | Bootstrap 上下文解析 |
| `src/agents/pi-embedded-helpers/bootstrap.ts` | Bootstrap 内容处理 |
| `src/agents/sanitize-for-prompt.ts` | 提示词安全清理 |

### 2.2 文档位置

- `docs/concepts/system-prompt.md` - System Prompt 概念文档

---

## 3. System Prompt 结构

### 3.1 主要组成部分

System Prompt 由以下部分组成（按顺序）：

```typescript
const lines = [
  "You are a personal assistant running inside OpenClaw.",
  "",                    // 基础身份
  "## Tooling",          // 工具列表和描述
  "## Safety",           // 安全指南
  "## Skills",           // 技能
  "## OpenClaw Self-Update",  // 自我更新
  "## Workspace",        // 工作目录
  "## Documentation",    // 文档
  "## Sandbox",          // 沙箱 (可选)
  "## Authorized Senders",  // 授权发送者
  "## Current Date & Time",  // 当前时间
  "## Workspace Files",  // 注入的工作区文件
  "## Reply Tags",       // 回复标签
  "## Messaging",        // 消息传递
  "## Voice (TTS)",     // 语音
  "## Silent Replies",  // 静默回复
  "## Heartbeats",      // 心跳
  "## Runtime",          // 运行时信息
  "## Reasoning Format", // 推理格式
];
```

### 3.2 Tooling 部分（核心工具）

```typescript
const coreToolSummaries: Record<string, string> = {
  read: "Read file contents",
  write: "Create or overwrite files",
  edit: "Make precise edits to files",
  apply_patch: "Apply multi-file patches",
  grep: "Search file contents for patterns",
  find: "Find files by glob pattern",
  ls: "List directory contents",
  exec: "Run shell commands (pty available for TTY-required CLIs)",
  process: "Manage background exec sessions",
  web_search: "Search the web (Brave API)",
  web_fetch: "Fetch and extract readable content from a URL",
  browser: "Control web browser",
  canvas: "Present/eval/snapshot the Canvas",
  nodes: "List/describe/notify/camera/screen on paired nodes",
  cron: "Manage cron jobs and wake events",
  message: "Send messages and channel actions",
  gateway: "Restart, apply config, or run updates",
  agents_list: "List agent ids allowed for sessions_spawn",
  sessions_list: "List other sessions",
  sessions_history: "Fetch history for another session",
  sessions_send: "Send a message to another session",
  sessions_spawn: "Spawn a sub-agent session",
  subagents: "List, steer, or kill sub-agent runs",
  session_status: "Show session status card",
  image: "Analyze an image with the configured image model",
};
```

### 3.3 工具顺序

```typescript
const toolOrder = [
  "read", "write", "edit", "apply_patch",
  "grep", "find", "ls", "exec", "process",
  "web_search", "web_fetch", "browser", "canvas",
  "nodes", "cron", "message", "gateway",
  "agents_list", "sessions_list", "sessions_history",
  "sessions_send", "subagents", "session_status", "image"
];
```

---

## 4. 提示词模式（Prompt Modes）

OpenClaw 支持三种提示词模式，通过 `PromptMode` 类型定义：

```typescript
export type PromptMode = "full" | "minimal" | "none";
```

### 4.1 模式说明

| 模式 | 描述 | 使用场景 |
|------|------|---------|
| **full** | 包含所有部分 | 主代理（默认） |
| **minimal** | 仅包含 Tooling、Workspace、Runtime | 子代理 |
| **none** | 仅基本身份行 | 最小化上下文 |

### 4.2 minimal 模式省略的部分

- Skills（技能）
- Memory Recall（记忆召回）
- Self-Update（自我更新）
- Authorized Senders（授权发送者）
- Reply Tags（回复标签）
- Messaging（消息传递）
- Voice (TTS)（语音）
- Documentation（文档）

---

## 5. Workspace Bootstrap 文件

### 5.1 默认 Bootstrap 文件

OpenClaw 使用一组预定义的 Bootstrap 文件来配置代理的身份和行为：

| 文件名 | 常量 | 描述 |
|--------|------|------|
| `AGENTS.md` | `DEFAULT_AGENTS_FILENAME` | 代理配置 |
| `SOUL.md` | `DEFAULT_SOUL_FILENAME` | 代理"灵魂"/核心个性 |
| `TOOLS.md` | `DEFAULT_TOOLS_FILENAME` | 工具配置 |
| `IDENTITY.md` | `DEFAULT_IDENTITY_FILENAME` | 身份定义 |
| `USER.md` | `DEFAULT_USER_FILENAME` | 用户信息 |
| `HEARTBEAT.md` | `DEFAULT_HEARTBEAT_FILENAME` | 心跳/定时任务配置 |
| `BOOTSTRAP.md` | `DEFAULT_BOOTSTRAP_FILENAME` | 启动引导 |
| `MEMORY.md` | `DEFAULT_MEMORY_FILENAME` | 记忆存储 |
| `memory.md` | `DEFAULT_MEMORY_ALT_FILENAME` | 记忆存储（备选） |

### 5.2 Workspace 目录位置

```typescript
// 默认工作目录
export function resolveDefaultAgentWorkspaceDir(): string {
  const home = os.homedir();
  const profile = process.env.OPENCLAW_PROFILE?.trim();
  if (profile && profile.toLowerCase() !== "default") {
    return path.join(home, ".openclaw", `workspace-${profile}`);
  }
  return path.join(home, ".openclaw", "workspace");
}
```

### 5.3 Bootstrap 文件加载

```typescript
export async function loadWorkspaceBootstrapFiles(dir: string): Promise<WorkspaceBootstrapFile[]> {
  const entries = [
    { name: DEFAULT_AGENTS_FILENAME, filePath: path.join(resolvedDir, DEFAULT_AGENTS_FILENAME) },
    { name: DEFAULT_SOUL_FILENAME, filePath: path.join(resolvedDir, DEFAULT_SOUL_FILENAME) },
    { name: DEFAULT_TOOLS_FILENAME, filePath: path.join(resolvedDir, DEFAULT_TOOLS_FILENAME) },
    { name: DEFAULT_IDENTITY_FILENAME, filePath: path.join(resolvedDir, DEFAULT_IDENTITY_FILENAME) },
    { name: DEFAULT_USER_FILENAME, filePath: path.join(resolvedDir, DEFAULT_USER_FILENAME) },
    { name: DEFAULT_HEARTBEAT_FILENAME, filePath: path.join(resolvedDir, DEFAULT_HEARTBEAT_FILENAME) },
    { name: DEFAULT_BOOTSTRAP_FILENAME, filePath: path.join(resolvedDir, DEFAULT_BOOTSTRAP_FILENAME) },
    // 记忆文件...
  ];
  // ...
}
```

### 5.4 最小化 Bootstrap 过滤

对于子代理和定时任务会话，只加载最小集合：

```typescript
const MINIMAL_BOOTSTRAP_ALLOWLIST = new Set([
  DEFAULT_AGENTS_FILENAME,
  DEFAULT_TOOLS_FILENAME,
  DEFAULT_SOUL_FILENAME,
  DEFAULT_IDENTITY_FILENAME,
  DEFAULT_USER_FILENAME,
]);

export function filterBootstrapFilesForSession(
  files: WorkspaceBootstrapFile[],
  sessionKey?: string,
): WorkspaceBootstrapFile[] {
  if (!sessionKey || (!isSubagentSessionKey(sessionKey) && !isCronSessionKey(sessionKey))) {
    return files;
  }
  return files.filter((file) => MINIMAL_BOOTSTRAP_ALLOWLIST.has(file.name));
}
```

---

## 6. 动态提示词生成逻辑

### 6.1 核心构建函数

```typescript
export function buildAgentSystemPrompt(params: {
  workspaceDir: string;
  defaultThinkLevel?: ThinkLevel;
  reasoningLevel?: ReasoningLevel;
  extraSystemPrompt?: string;
  ownerNumbers?: string[];
  ownerDisplay?: OwnerIdDisplay;
  ownerDisplaySecret?: string;
  reasoningTagHint?: boolean;
  toolNames?: string[];
  toolSummaries?: Record<string, string>;
  modelAliasLines?: string[];
  userTimezone?: string;
  userTime?: string;
  userTimeFormat?: ResolvedTimeFormat;
  contextFiles?: EmbeddedContextFile[];
  skillsPrompt?: string;
  heartbeatPrompt?: string;
  docsPath?: string;
  workspaceNotes?: string[];
  ttsHint?: string;
  promptMode?: PromptMode;
  runtimeInfo?: RuntimeInfo;
  messageToolHints?: string[];
  sandboxInfo?: EmbeddedSandboxInfo;
  reactionGuidance?: ReactionGuidance;
  memoryCitationsMode?: MemoryCitationsMode;
}): string {
  // 构建逻辑...
}
```

### 6.2 Skills 提示词构建

```typescript
export function buildWorkspaceSkillsPrompt(
  workspaceDir: string,
  opts?: WorkspaceSkillBuildOptions,
): string {
  // 1. 加载 Skills
  const skillEntries = loadSkillEntries(workspaceDir, opts);

  // 2. 过滤符合条件的 Skills
  const eligible = filterSkillEntries(skillEntries, ...);

  // 3. 应用字符限制
  const { skillsForPrompt, truncated } = applySkillsPromptLimits(...);

  // 4. 生成格式化的 Skills 提示词
  const prompt = formatSkillsForPrompt(compactSkillPaths(skillsForPrompt));
  return prompt;
}
```

### 6.3 Skills 格式（XML 标签）

Skills 使用 XML 格式注入提示词：

```xml
<available_skills>
  <skill>
    <name>...</name>
    <description>...</description>
    <location>...</location>
  </skill>
</available_skills>
```

### 6.4 Skills 匹配指令

```typescript
function buildSkillsSection(params: { skillsPrompt?: string; readToolName: string }) {
  return [
    "## Skills (mandatory)",
    "Before replying: scan <available_skills> <description> entries.",
    `- If exactly one skill clearly applies: read its SKILL.md at <location> with \`${params.readToolName}\`, then follow it.`,
    "- If multiple could apply: choose the most specific one, then read/follow it.",
    "- If none clearly apply: do not read any SKILL.md.",
    "Constraints: never read more than one skill up front; only read after selecting.",
    trimmed,
    "",
  ];
}
```

---

## 7. 提示词安全性

### 7.1 安全清理函数

防止提示词注入攻击：

```typescript
export function sanitizeForPromptLiteral(value: string): string {
  // 移除 Unicode 控制字符 (Cc) 和格式字符 (Cf)
  // 移除行/段落分隔符 (U+2028/U+2029)
  return value.replace(/[\p{Cc}\p{Cf}\u2028\u2029]/gu, "");
}
```

### 7.2 所有者身份哈希

保护用户隐私，对所有者 ID 进行哈希处理：

```typescript
function formatOwnerDisplayId(ownerId: string, ownerDisplaySecret?: string) {
  const hasSecret = ownerDisplaySecret?.trim();
  const digest = hasSecret
    ? createHmac("sha256", hasSecret).update(ownerId).digest("hex")
    : createHash("sha256").update(ownerId).digest("hex");
  return digest.slice(0, 12);
}

function buildOwnerIdentityLine(
  ownerNumbers: string[],
  ownerDisplay: OwnerIdDisplay,
  ownerDisplaySecret?: string,
) {
  const displayOwnerNumbers =
    ownerDisplay === "hash"
      ? normalized.map((ownerId) => formatOwnerDisplayId(ownerId, ownerDisplaySecret))
      : normalized;
  return `Authorized senders: ${displayOwnerNumbers.join(", ")}.`;
}
```

---

## 8. Session Reset Prompt

当用户执行 `/new` 或 `/reset` 时，会注入特定的启动提示词：

```typescript
export const BARE_SESSION_RESET_PROMPT =
  "A new session was started via /new or /reset. " +
  "Execute your Session Startup sequence now - read the required files before responding to the user. " +
  "Then greet the user in your configured persona, if one is provided. " +
  "Be yourself - use your defined voice, mannerisms, and mood. " +
  "Keep it to 1-3 sentences and ask what they want to do. " +
  "If the runtime model differs from default_model in the system prompt, mention the default model. " +
  "Do not mention internal steps, files, tools, or reasoning.";
```

---

## 9. 通道差异化提示词

### 9.1 通道能力配置

不同消息通道支持不同的功能：

```typescript
export function resolveChannelCapabilities(params: {
  cfg?: Partial<OpenClawConfig>;
  channel?: string | null;
  accountId?: string | null;
}): string[] | undefined {
  // 解析通道配置中的 capabilities
}
```

### 9.2 内联按钮支持

根据通道能力生成不同的提示词：

```typescript
const inlineButtonsEnabled = runtimeCapabilitiesLower.has("inlinebuttons");

// 启用时
"- Inline buttons supported. Use `action=send` with `buttons=[[{text,callback_data,style?}]]`; `style` can be `primary`, `success`, or `danger`."

// 禁用时
`- Inline buttons not enabled for ${params.runtimeChannel}. If you need them, ask to set ${params.runtimeChannel}.capabilities.inlineButtons ("dm"|"group"|"all"|"allowlist").`
```

### 9.3 群组聊天提示词

不同群组通道有不同的提示词模板：

```typescript
// Discord 特定
`You are in the Discord group chat "Release Squad". Participants: Alice, Bob.`,
`Activation: trigger-only (you are invoked only when explicitly mentioned; recent context may be included). ${groupParticipationNote}`,

// WhatsApp 特定
`You are in the WhatsApp group chat "Ops".`,
`WhatsApp IDs: SenderId is the participant JID (group participant id).`,

// Telegram 特定
`You are in the Telegram group chat "Dev Chat".`,
```

### 9.4 群组参与指南

```typescript
const groupParticipationNote =
  "Be a good group participant: mostly lurk and follow the conversation; " +
  "reply only when directly addressed or you can add clear value. " +
  "Emoji reactions are welcome when available. " +
  "Write like a human. " +
  "Avoid Markdown tables. " +
  "Don't type literal \\n sequences; use real line breaks sparingly.";
```

---

## 10. 消息传递部分详解

### 10.1 消息工具使用指南

```typescript
function buildMessagingSection(params: {
  isMinimal: boolean;
  availableTools: Set<string>;
  messageChannelOptions: string;
  inlineButtonsEnabled: boolean;
  runtimeChannel?: string;
  messageToolHints?: string[];
}) {
  return [
    "## Messaging",
    "- Reply in current session → automatically routes to the source channel (Signal, Telegram, etc.)",
    "- Cross-session messaging → use sessions_send(sessionKey, message)",
    "- Sub-agent orchestration → use subagents(action=list|steer|kill)",
    "- `[System Message] ...` blocks are internal context and are not user-visible by default.",
    `- If a \`[System Message]\` reports completed cron/subagent work and asks for a user update, rewrite it in your normal assistant voice and send that update (do not forward raw system text or default to ${SILENT_REPLY_TOKEN}).`,
    "- Never use exec/curl for provider messaging; OpenClaw handles all routing internally.",
    // 工具相关...
  ];
}
```

---

## 11. 内存召回部分

### 11.1 记忆搜索指令

```typescript
function buildMemorySection(params: {
  isMinimal: boolean;
  availableTools: Set<string>;
  citationsMode?: MemoryCitationsMode;
}) {
  const lines = [
    "## Memory Recall",
    "Before answering anything about prior work, decisions, dates, people, preferences, or todos: " +
    "run memory_search on MEMORY.md + memory/*.md; " +
    "then use memory_get to pull only the needed lines. " +
    "If low confidence after search, say you checked.",
  ];
  if (params.citationsMode === "off") {
    lines.push(
      "Citations are disabled: do not mention file paths or line numbers in replies unless the user explicitly asks.",
    );
  } else {
    lines.push(
      "Citations: include Source: <path#line> when it helps the user verify memory snippets.",
    );
  }
  return lines;
}
```

---

## 12. 回复标签（Reply Tags）

```typescript
function buildReplyTagsSection(isMinimal: boolean) {
  return [
    "## Reply Tags",
    "To request a native reply/quote on supported surfaces, include one tag in your reply:",
    "- Reply tags must be the very first token in the message (no leading text/newlines): [[reply_to_current]] your reply.",
    "- [[reply_to_current]] replies to the triggering message.",
    "- Prefer [[reply_to_current]]. Use [[reply_to:<id>]] only when an id was explicitly provided (e.g. by the user or a tool).",
    "Whitespace inside the tag is allowed (e.g. [[ reply_to_current ]] / [[ reply_to: 123 ]]).",
    "Tags are stripped before sending; support depends on the current channel config.",
    "",
  ];
}
```

---

## 13. 文档部分

```typescript
function buildDocsSection(params: { docsPath?: string; isMinimal: boolean; readToolName: string }) {
  return [
    "## Documentation",
    `OpenClaw docs: ${docsPath}`,
    "Mirror: https://docs.openclaw.ai",
    "Source: https://github.com/openclaw/openclaw",
    "Community: https://discord.com/invite/clawd",
    "Find new skills: https://clawhub.com",
    "For OpenClaw behavior, commands, config, or architecture: consult local docs first.",
    "When diagnosing issues, run `openclaw status` yourself when possible; only ask the user if you lack access (e.g., sandboxed).",
    "",
  ];
}
```

---

## 14. 运行时信息

```typescript
runtimeInfo?: {
  agentId?: string;
  host?: string;
  os?: string;
  arch?: string;
  node?: string;
  model?: string;
  defaultModel?: string;
  shell?: string;
  channel?: string;
  capabilities?: string[];
  repoRoot?: string;
};
```

---

## 15. 调用链示例

```typescript
// 1. 从配置构建参数
const { runtimeInfo, userTimezone, userTime, userTimeFormat } = buildSystemPromptParams({
  config: params.cfg,
  agentId: sessionAgentId,
  workspaceDir,
  runtime: { host, os, arch, node, model, defaultModel },
});

// 2. 构建 Skills Prompt
const skillsPrompt = buildWorkspaceSkillsPrompt(workspaceDir, { config: params.cfg });

// 3. 构建 Bootstrap Context
const { bootstrapFiles, contextFiles } = await resolveBootstrapContextForRun({
  workspaceDir,
  config: params.cfg,
  sessionKey,
});

// 4. 构建完整的 System Prompt
const systemPrompt = buildAgentSystemPrompt({
  workspaceDir,
  runtimeInfo,
  userTimezone,
  skillsPrompt,
  contextFiles,
  // ... 其他参数
});
```

---

## 16. 字符限制

### 16.1 Bootstrap 上下文限制

```typescript
const DEFAULT_BOOTSTRAP_MAX_CHARS = 20_000;      // 单个文件最大字符数
const DEFAULT_BOOTSTRAP_TOTAL_MAX_CHARS = 150_000;  // 总最大字符数
```

### 16.2 截断策略

当内容超过限制时，系统会：
1. 对单个文件应用字符限制
2. 对总内容应用全局限制
3. 优先保留较早/重要的内容

---

## 17. 总结

OpenClaw 的提示词工程具有以下特点：

| 特性 | 说明 |
|------|------|
| **模块化设计** | 提示词由多个独立的部分组成，每个部分通过专门的函数构建 |
| **动态生成** | 所有提示词内容根据运行时参数动态生成 |
| **安全性** | 使用 `sanitizeForPromptLiteral` 函数清理用户输入，防止提示词注入 |
| **上下文注入** | 通过 bootstrap 文件机制，将用户配置文件注入到提示词中 |
| **通道差异** | 针对不同消息通道（Discord、Telegram、WhatsApp 等）生成特定的提示词内容 |
| **模式支持** | 支持 full、minimal、none 三种提示词模式，适用于不同场景 |
| **记忆系统** | 内置 Memory Recall 功能，支持向量搜索和引用 |
| **技能匹配** | Skills 使用 XML 格式注入，代理自动匹配并加载相关技能 |

---

*本文档由 AI 生成，基于对 OpenClaw 项目源代码的分析。*
