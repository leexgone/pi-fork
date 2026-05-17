# 提示词与上下文管理设计文档

Pi 项目的提示词系统和上下文管理系统是 AI 代理的核心引擎。本文档详细分析系统提示词构建、扩展注入、会话存储、上下文压缩、分支管理等机制。

## 目录

- [系统提示词构建](#系统提示词构建)
- [提示词组装管线](#提示词组装管线)
- [自定义提示词文件](#自定义提示词文件)
- [扩展动态注入](#扩展动态注入)
- [技能系统](#技能系统)
- [消息类型体系](#消息类型体系)
- [消息转换管线](#消息转换管线)
- [会话存储与树结构](#会话存储与树结构)
- [上下文压缩系统](#上下文压缩系统)
- [溢出检测与恢复](#溢出检测与恢复)
- [分支摘要管理](#分支摘要管理)
- [完整上下文生命周期](#完整上下文生命周期)

---

## 系统提示词构建

核心函数：`buildSystemPrompt()` 位于 [packages/coding-agent/src/core/system-prompt.ts:28-172](../packages/coding-agent/src/core/system-prompt.ts#L28-L172)

### 输入参数

```typescript
interface BuildSystemPromptOptions {
    customPrompt?: string;                          // 完全替换默认提示词
    selectedTools?: string[];                       // 要包含的工具列表
    toolSnippets?: Record<string, string>;          // 工具一句话描述
    promptGuidelines?: string[];                    // 附加指南条目
    appendSystemPrompt?: string;                    // 追加到系统提示词的文本
    cwd: string;                                    // 工作目录
    contextFiles?: Array<{path, content}>;          // 预加载上下文文件
    skills?: Skill[];                               // 预加载技能
}
```

### 两条构建路径

**路径 A — 自定义提示词**：若 `customPrompt` 有值，以此为基础，追加：
1. `appendSystemPrompt` 段
2. `# Project Context` 上下文文件
3. Skills 段（仅当 `read` 工具可用时）
4. 当前日期和工作目录

**路径 B — 默认提示词**：完整构建内置模板：

```
You are an expert coding assistant operating inside pi, a coding agent harness.
You help users by reading files, executing commands, editing code, and writing new files.

Available tools:
- read: Read file contents
- bash: Execute bash commands (ls, grep, find, etc.)
- edit: Make precise file edits with exact text replacement...
- write: Create or overwrite files

In addition to the tools above, you may have access to other custom tools depending on the project.

Guidelines:
- Prefer grep/find/ls tools over bash for file exploration (faster, respects .gitignore)
- Use write only for new files or complete rewrites.
- Be concise in your responses
- Show file paths clearly when working with files

Pi documentation (read only when the user asks about pi itself...):
- Main documentation: /path/to/README.md
- Additional docs: /path/to/docs
- Examples: /path/to/examples
- ...

Current date: 2026-05-17
Current working directory: /path/to/project
```

### 智能指南自适应

指南部分根据可用工具集动态调整（[system-prompt.ts:111-116](../packages/coding-agent/src/core/system-prompt.ts#L111-L116)）：

| 可用工具 | 自动添加的指南 |
|---------|-------------|
| 仅 `bash` | "Use bash for file operations like ls, rg, find" |
| `bash` + `grep`/`find`/`ls` | "Prefer grep/find/ls tools over bash for file exploration" |

---

## 提示词组装管线

`AgentSession._rebuildSystemPrompt()` 位于 [packages/coding-agent/src/core/agent-session.ts:918-951](../packages/coding-agent/src/core/agent-session.ts#L918-L951)，是中央组装点。

### 组装步骤

```
1. 遍历活跃工具名
   → 从 _toolPromptSnippets 收集 promptSnippet
   → 从 _toolPromptGuidelines 收集 promptGuidelines（去重）

2. 读取资源加载器的 SYSTEM.md（如存在则替换默认提示词）
3. 读取 APPEND_SYSTEM.md（如存在则追加到提示词）

4. 加载 Skills 和 Context Files

5. 构建 BuildSystemPromptOptions → 调用 buildSystemPrompt()

6. 存储到 this._baseSystemPrompt 和 this.agent.state.systemPrompt
```

### 工具提示元数据来源

内置工具在定义时携带提示元数据：

| 工具 | promptSnippet | promptGuidelines |
|------|--------------|------------------|
| **read** | "Read file contents" | ["Use read to examine files instead of cat or sed."] |
| **bash** | "Execute bash commands (ls, grep, find, etc.)" | — |
| **edit** | "Make precise file edits..." | 4 条使用指南（匹配策略、多次编辑等） |
| **write** | "Create or overwrite files" | ["Use write only for new files or complete rewrites."] |
| **grep** | "Search file contents for patterns (respects .gitignore)" | — |
| **find** | "Find files by glob pattern (respects .gitignore)" | — |
| **ls** | "List directory contents" | — |

---

## 自定义提示词文件

### SYSTEM.md / APPEND_SYSTEM.md

由资源加载器发现（[packages/coding-agent/src/core/resource-loader.ts:844-870](../packages/coding-agent/src/core/resource-loader.ts#L844-L870)）：

| 文件 | 查找路径 | 作用 |
|------|---------|------|
| **SYSTEM.md** | `{cwd}/.pi/SYSTEM.md` 或 `~/.pi/agent/SYSTEM.md` | **完全替换**默认系统提示词 |
| **APPEND_SYSTEM.md** | `{cwd}/.pi/APPEND_SYSTEM.md` 或 `~/.pi/agent/APPEND_SYSTEM.md` | **追加**到系统提示词末尾 |

### 项目上下文文件（AGENTS.md / CLAUDE.md）

加载函数 `loadProjectContextFiles()`（[resource-loader.ts:58-114](../packages/coding-agent/src/core/resource-loader.ts#L58-L114)）：

1. 先加载全局 `~/.pi/agent/AGENTS.md`
2. 从 cwd 向上遍历到根目录，收集每层的 `AGENTS.md` / `CLAUDE.md`
3. 按从根到 cwd 的顺序插入，**越靠近项目的文件越靠后**（优先级更高）
4. 内容追加到系统提示词的 `# Project Context` 段下

### 用户提示词模板

`.pi/prompts/` 目录下存放用户可调用的模板（非系统提示词组件），通过 `/name` 命令展开：

- `cl.md` — 完整收尾（变更日志 + 提交 + 推送）
- `is.md` — GitHub Issue 分析
- `pr.md` — PR 审查（结构化输出）
- `wr.md` — 发布前变更日志审计

使用 YAML frontmatter 定义 `description` 和 `argument-hint`，支持参数替换（`$1`、`$@`、`$ARGUMENTS`）。

---

## 扩展动态注入

### before_agent_start 事件

每次 LLM 调用前触发（[packages/coding-agent/src/core/extensions/runner.ts:924-988](../packages/coding-agent/src/core/extensions/runner.ts#L924-L988)）：

```typescript
interface BeforeAgentStartEvent {
    type: "before_agent_start";
    prompt: string;                  // 用户输入
    images?: ImageContent[];
    systemPrompt: string;            // 当前系统提示词
    systemPromptOptions: BuildSystemPromptOptions;
}

interface BeforeAgentStartEventResult {
    message?: { role: string; content: string };  // 注入用户消息
    systemPrompt?: string;                         // 替换系统提示词
}
```

扩展可以：
- 返回 `{ systemPrompt: "..." }` 完全替换当前提示词
- 返回 `{ message: { role: "user", content: "..." } }` 注入用户消息
- 多个处理器链式执行，后续处理器看到前一个处理器的结果

典型示例：`pirate.ts` 扩展追加海盗模式指令；`prompt-customizer.ts` 读取 `systemPromptOptions` 添加工具特定指导。

---

## 技能系统

### 技能发现

位于 [packages/coding-agent/src/core/skills.ts](../packages/coding-agent/src/core/skills.ts)：

1. **全局技能**: `~/.pi/agent/skills/` 目录
2. **项目技能**: `<cwd>/.pi/skills/`
3. **显式路径**: `--skill` 参数指定

发现规则：
- 目录中包含 `SKILL.md` 文件即视为技能根目录
- 遵守 `.gitignore` / `.ignore` / `.fdignore`
- 解析 YAML frontmatter 获取 `name`、`description`、`disable-model-invocation`

### 系统提示词中的技能展示

通过 `formatSkillsForPrompt()`（[skills.ts:336-362](../packages/coding-agent/src/core/skills.ts#L336-L362)）格式化为 XML：

```xml
The following skills provide specialized instructions for specific tasks.
Use the read tool to load a skill's file when the task matches its description.

<available_skills>
  <skill>
    <name>skill-name</name>
    <description>Skill description</description>
    <location>/absolute/path/to/SKILL.md</location>
  </skill>
</available_skills>
```

标记为 `disable-model-invocation: true` 的技能不会出现在提示词中（只能通过 `/skill:name` 命令调用）。

---

## 消息类型体系

Pi 使用 TypeScript **声明合并**（declaration merging）扩展基础消息类型。

### 基础消息类型

定义在 [packages/ai/src/types.ts:271-302](../packages/ai/src/types.ts#L271-L302)：

```typescript
// 用户消息
interface UserMessage {
    role: "user";
    content: string | (TextContent | ImageContent)[];
}

// 助手消息
interface AssistantMessage {
    role: "assistant";
    content: (TextContent | ThinkingContent | ToolCall)[];
    usage?: { input: number; output: number; ... };
    stopReason?: string;
}

// 工具结果消息
interface ToolResultMessage {
    role: "toolResult";
    toolCallId: string;
    toolName: string;
    content: (TextContent | ImageContent)[];
    details?: unknown;
    isError: boolean;
    timestamp: number;
}
```

### 扩展消息类型

通过声明合并添加自定义类型（[packages/coding-agent/src/core/messages.ts:69-77](../packages/coding-agent/src/core/messages.ts#L69-L77)）：

```typescript
declare module "@earendil-works/pi-agent-core" {
    interface CustomAgentMessages {
        bashExecution: BashExecutionMessage;    // ! 命令的 bash 输出
        custom: CustomMessage;                   // 扩展注入的消息
        branchSummary: BranchSummaryMessage;     // 分支探索摘要
        compactionSummary: CompactionSummaryMessage; // 压缩摘要
    }
}
```

各类型详情：

**BashExecutionMessage**（[messages.ts:29-40](../packages/coding-agent/src/core/messages.ts#L29-L40)）：
```typescript
{
    role: "bashExecution";
    command: string;
    output: string;
    exitCode: number | undefined;
    cancelled: boolean;
    truncated: boolean;
    fullOutputPath?: string;        // 截断时完整输出的文件路径
    timestamp: number;
    excludeFromContext?: boolean;   // !! 前缀命令排除出上下文
}
```

**CustomMessage**（[messages.ts:46-53](../packages/coding-agent/src/core/messages.ts#L46-L53)）：
```typescript
{
    role: "custom";
    customType: string;             // 扩展自定义的类型标识
    content: string | (TextContent | ImageContent)[];
    display: boolean;               // 是否在 UI 中显示
    details?: unknown;
    timestamp: number;
}
```

**BranchSummaryMessage**（[messages.ts:55-59](../packages/coding-agent/src/core/messages.ts#L55-L59)）：
```typescript
{
    role: "branchSummary";
    summary: string;
    fromId: string;    // 来源分支入口 ID
    timestamp: number;
}
```

**CompactionSummaryMessage**（[messages.ts:62-67](../packages/coding-agent/src/core/messages.ts#L62-L67)）：
```typescript
{
    role: "compactionSummary";
    summary: string;
    tokensBefore: number;  // 压缩前的 token 数
    timestamp: number;
}
```

### AgentMessage 联合类型

```typescript
// ../packages/agent/src/types.ts:309
export type AgentMessage = Message | CustomAgentMessages[keyof CustomAgentMessages];
```

---

## 消息转换管线

`convertToLlm()` 位于 [packages/coding-agent/src/core/messages.ts:148-195](../packages/coding-agent/src/core/messages.ts#L148-L195)，将 `AgentMessage[]` 转换为 LLM 可接受的 `Message[]`。

### 转换规则

| 消息类型 | 转换方式 |
|---------|---------|
| **bashExecution**（排除） | `excludeFromContext=true` → 过滤掉（`!!` 前缀） |
| **bashExecution**（保留） | → User 消息，格式为 `Ran \`cmd\`\n\`\`\`output\`\`\`[exit code]` |
| **custom** | → User 消息，content 直接传递 |
| **branchSummary** | → User 消息，content 用 `<summary>` XML 标签包裹 |
| **compactionSummary** | → User 消息，content 用 `<summary>` XML 标签包裹 |
| **user/assistant/toolResult** | 原样传递 |

XML 包裹前缀/后缀：

```typescript
const COMPACTION_SUMMARY_PREFIX = "The conversation history before this point was compacted into the following summary:\n\n<summary>\n";
const COMPACTION_SUMMARY_SUFFIX = "\n</summary>";

const BRANCH_SUMMARY_PREFIX = "The following is a summary of a branch that this conversation came back from:\n\n<summary>\n";
const BRANCH_SUMMARY_SUFFIX = "</summary>";
```

### Agent Loop 中的转换点

在 `AgentLoop` 每次调用 LLM 前（[agent-loop.ts:275-368](../packages/agent/src/agent-loop.ts#L275-L368)）：

```typescript
// 应用上下文转换（可自定义）
let messages = context.messages;
if (config.transformContext) {
    messages = await config.transformContext(messages, signal);
}
// 转换为 LLM 兼容格式
const llmMessages = await config.convertToLlm(messages);
// 调用 LLM
streamSimple({ messages: llmMessages, systemPrompt: context.systemPrompt, ... })
```

---

## 会话存储与树结构

### SessionManager

位于 [packages/coding-agent/src/core/session-manager.ts](../packages/coding-agent/src/core/session-manager.ts)。

**存储格式**: JSONL（每行一个 JSON 对象），文件头是 Session 元数据，后续每行是 Entry。

**树结构**: 每个 Entry 有 `id` 和 `parentId`，形成一棵消息树。`leafId` 指针标记当前对话位置。

**Entry 类型**:

| 类型 | 说明 |
|------|------|
| `message` | 用户/助手/工具结果消息 |
| `thinking_level_change` | 思考级别变更记录 |
| `model_change` | 模型切换记录 |
| `compaction` | 压缩操作（含摘要、保留消息范围） |
| `branch_summary` | 分支探索摘要 |
| `custom` | 自定义条目 |
| `custom_message` | 自定义消息 |
| `label` | 会话标签 |
| `session_info` | 会话元信息 |

### 会话上下文构建

`buildSessionContext()`（[session-manager.ts:314-421](../packages/coding-agent/src/core/session-manager.ts#L314-L421)）从叶子到根遍历消息树：

```
1. 从 leafId 沿 parentId 链回溯到根
2. 提取路径上的 thinkingLevel 和 model
3. 找到最近的 compaction 条目
4. 如果存在压缩：
   a. 先发射压缩摘要消息
   b. 包含 firstKeptEntryId 到 compaction 之间的保留消息
   c. 再包含压缩后的所有消息
5. 转换 custom_message 和 branch_summary 为 AgentMessage 格式
6. 返回 { messages, thinkingLevel, model }
```

---

## 上下文压缩系统

### 触发条件

位于 [packages/coding-agent/src/core/compaction/compaction.ts:219-222](../packages/coding-agent/src/core/compaction/compaction.ts#L219-L222)：

```typescript
export function shouldCompact(contextTokens: number, contextWindow: number, settings): boolean {
    if (!settings.enabled) return false;
    return contextTokens > contextWindow - settings.reserveTokens;
}
```

**默认设置**（[compaction.ts:115-125](../packages/coding-agent/src/core/compaction/compaction.ts#L115-L125)）：

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `enabled` | true | 是否启用压缩 |
| `reserveTokens` | 16384 | 为提示词和响应预留的 token |
| `keepRecentTokens` | 20000 | 保留的最近消息 token 数 |

以 128K 模型为例：当 `contextTokens > 128000 - 16384 = 111616` 时触发压缩。

### Token 估算

使用 `chars / 4` 启发式方法（保守高估）：

- **User 消息**: 累加文本内容字符数
- **Assistant 消息**: text + thinking + tool call (name + JSON args)
- **ToolResult/Custom**: 文本字符数，图片估算为 4800 chars（1200 tokens）
- **BashExecution**: command.length + output.length

`estimateContextTokens()` 找到最后一个有 usage 数据的 assistant 消息作为锚点，使用该锚点的实际 token 计数加上其后消息的估算值。

### 压缩准备

`prepareCompaction()`（[compaction.ts:617-692](../packages/coding-agent/src/core/compaction/compaction.ts#L617-L692)）：

```
1. 找到上次压缩条目（边界标记）→ boundaryStart
2. 估算当前总 token 数
3. findCutPoint() 确定切割点
4. 提取 messagesToSummarize（boundaryStart 到切割点之间的消息）
5. 提取 turnPrefixMessages（如果切割在 turn 中间）
6. 提取文件操作列表用于摘要
```

### 切割点检测

`findCutPoint()`（[compaction.ts:386-448](../packages/coding-agent/src/core/compaction/compaction.ts#L386-L448)）：

1. 从最新消息向后遍历，累加 token 估算值
2. 当累加值 ≥ `keepRecentTokens` 时，找到最接近的有效切割点
3. 切割点不能是 toolResult 消息（必须是 user/assistant/custom/bashExecution）
4. 如果切割在 assistant 消息处，向前找到 turn 起点（前一个 user/bashExecution 消息）作为 turn 前缀

### 摘要生成

`generateSummary()`（[compaction.ts:530-593](../packages/coding-agent/src/core/compaction/compaction.ts#L530-L593)）：

**防对话陷阱**: 先将对话序列化为纯文本格式（`serializeConversation`），防止模型将历史当作正在进行的对话来回应。

```
[User]: <text>
[Assistant]: <text>
[Assistant thinking]: <thinking>
[Assistant tool calls]: read(path="..."); edit(path="...")
[Tool result]: <truncated text>
```

工具结果截断为 2000 字符以提高摘要效率。

**摘要系统提示词**（[compaction/utils.ts:168](../packages/coding-agent/src/core/compaction/utils.ts#L168)）：

```
You are a context summarization assistant. Your task is to read a conversation
between a user and an AI coding assistant, then produce a structured summary
following the exact format specified.

Do NOT continue the conversation. Do NOT respond to any questions in the
conversation. ONLY output the structured summary.
```

**摘要格式**: Goal / Constraints / Progress (Done/In Progress/Blocked) / Key Decisions / Next Steps / Critical Context。

如果已有上次压缩的摘要，会使用**更新提示词**（[compaction.ts:487-524](../packages/coding-agent/src/core/compaction/compaction.ts#L487-L524)），将新信息合并到现有内容中。

### 文件操作追踪

`compaction/utils.ts`（[compaction/utils.ts:12-82](../packages/coding-agent/src/core/compaction/utils.ts#L12-L82)）从工具调用中提取被读取/修改的文件列表，以 XML 格式追加到摘要：

```xml
<read-files>path/to/file.ts</read-files>
<modified-files>path/to/file.ts</modified-files>
```

### 完整压缩流程

`compact()`（[compaction.ts:720-800](../packages/coding-agent/src/core/compaction/compaction.ts#L720-L800)）：

```
prepareCompaction() → 找到切割点和待摘要消息
    ↓
如果是 split turn → 并行生成主历史摘要 + turn 前缀摘要（Promise.all）
    ↓
生成主摘要（和 turn 前缀摘要）
    ↓
合并两个摘要
    ↓
创建 CompactionSummaryMessage → 保存到 SessionManager
    ↓
压缩后的会话 = 摘要 + 保留的最近消息 + 压缩后消息
```

---

## 溢出检测与恢复

### 溢出检测

位于 [packages/ai/src/utils/overflow.ts:33-151](../packages/ai/src/utils/overflow.ts#L33-L151)：

`isContextOverflow()` 处理三种检测方式：

1. **错误匹配**: 匹配 20+ 提供商的上下文溢出错误模式（Anthropic、OpenAI、Google、xAI、Groq、OpenRouter、Together AI 等）
2. **静默溢出（z.ai 风格）**: `stopReason === "stop"` 但 `usage.input + usage.cacheRead > contextWindow`
3. **长度停止溢出（小米 MiMo 风格）**: `stopReason === "length"` 且 `output === 0` 且输入占满 ~99% contextWindow

同时排除节流、限流等非溢出模式。

### 溢出恢复

`AgentSession` 的溢出处理（[agent-session.ts:1793-1815](../packages/coding-agent/src/core/agent-session.ts#L1793-L1815)）：

```
检测到溢出 → 检查 _overflowRecoveryAttempted 防护（仅一次重试）
    ↓
从 Agent 状态移除错误消息
    ↓
运行 _runAutoCompaction("overflow", true)
    ↓
压缩完成后延迟 100ms → agent.continue() 重试
```

---

## 分支摘要管理

### 会话树分支

`AgentSessionRuntime.fork()`（[agent-session-runtime.ts:234-320](../packages/coding-agent/src/core/agent-session-runtime.ts#L234-L320)）：

```
选择分支入口点 → 定位到目标 leafId
    ↓
sessionManager.createBranchedSession(targetLeafId) → 创建分支会话文件
    ↓
销毁旧运行时 → 创建新运行时
```

### 分支摘要生成

当导航离开分支时，`generateBranchSummary()`（[compaction/branch-summarization.ts](../packages/coding-agent/src/core/compaction/branch-summarization.ts)）：

1. `collectEntriesForBranchSummary()`: 找到旧叶子与新目标的共同祖先，回溯收集所有条目
2. `prepareBranchEntries()`: 按 token 预算从新到旧遍历，优先保留摘要条目
3. 生成结构化摘要：Goal / Constraints / Progress / Key Decisions / Next Steps

导航发生时，`sessionManager.moveTo(entryId, summary)` 在目标叶子位置追加 `branch_summary` 条目，保留被放弃分支的上下文。

---

## 完整上下文生命周期

```
┌─ 会话创建 ──────────────────────────────────────────────────────┐
│                                                                 │
│  AgentSession 创建                                               │
│    → _rebuildSystemPrompt() 构建基础系统提示词                    │
│    → 加载 SYSTEM.md / APPEND_SYSTEM.md                           │
│    → 加载 AGENTS.md / CLAUDE.md 上下文文件                        │
│    → 加载 Skills                                                 │
│    → 收集工具 promptSnippet + promptGuidelines                   │
│                                                                 │
├─ 每次 LLM 调用前 ───────────────────────────────────────────────┤
│                                                                 │
│  emitBeforeAgentStart() → 扩展可修改系统提示词                    │
│  convertToLlm() → AgentMessage[] → Message[]                    │
│  streamSimple() → LLM 流式响应                                   │
│                                                                 │
├─ 对话进行中 ────────────────────────────────────────────────────┤
│                                                                 │
│  消息追加到 SessionManager 树                                     │
│  每次 agent_end 后检查是否需要压缩:                               │
│    ├─ 溢出检测 → isContextOverflow()                             │
│    │   → 压缩 → 生成摘要 → 保留最近消息 → 自动重试                │
│    └─ 阈值检测 → shouldCompact()                                 │
│        → 压缩 → 生成摘要 → 保留最近消息                           │
│                                                                 │
├─ 分支导航 ──────────────────────────────────────────────────────┤
│                                                                 │
│  fork() → 创建分支                                               │
│  离开旧分支 → generateBranchSummary()                            │
│    → 追加 branch_summary 到新叶子                                 │
│                                                                 │
├─ 压缩后恢复 ────────────────────────────────────────────────────┤
│                                                                 │
│  buildSessionContext() 重建消息:                                 │
│    1. 压缩摘要（<summary> 包裹）                                  │
│    2. 保留的消息（firstKeptEntryId → compaction）                │
│    3. 压缩后的新消息                                              │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```
