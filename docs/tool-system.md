# 工具系统设计文档

Pi 项目的工具系统为 AI 代理提供了与代码、文件和系统交互的能力。本文档详细描述了工具的架构设计、接口定义、执行流程和扩展机制。

## 目录

- [架构概览](#架构概览)
- [三层接口体系](#三层接口体系)
- [内置工具](#内置工具)
- [工具执行管线](#工具执行管线)
- [截断系统](#截断系统)
- [文件变异队列](#文件变异队列)
- [注册与组合](#注册与组合)
- [事件与 Hook](#事件与-hook)
- [TUI 渲染](#tui-渲染)
- [Read 工具图片处理](#read-工具图片处理)
- [扩展自定义工具](#扩展自定义工具)

---

## 架构概览

工具系统横跨三个包，采用**分层抽象**设计：

```
┌─────────────────────────────────────────────────────────┐
│  Extension Layer (coding-agent)                          │
│  ToolDefinition  ── prompt metadata, TUI renderers       │
├─────────────────────────────────────────────────────────┤
│  Bridge Layer (tool-definition-wrapper)                  │
│  wrapToolDefinition()  ── 转换适配                        │
├─────────────────────────────────────────────────────────┤
│  Core Layer (agent)                                      │
│  AgentTool  ── execute, prepareArguments, events         │
├─────────────────────────────────────────────────────────┤
│  AI Layer (ai)                                           │
│  Tool  ── name, description, TypeBox parameters          │
└─────────────────────────────────────────────────────────┘
```

- **AI 层** (`packages/ai/src/types.ts`): 定义最基础的 `Tool` 接口，包含 `name`、`description` 和 TypeBox 参数 schema。
- **Agent 核心层** (`packages/agent/src/types.ts`): 扩展为 `AgentTool`，增加 `execute()` 方法、`prepareArguments()` 钩子、执行模式等运行时能力。
- **编码代理层** (`packages/coding-agent/src/core/extensions/types.ts`): 进一步扩展为 `ToolDefinition`，增加系统提示片段、TUI 渲染器、`ExtensionContext` 注入等。

每一层都向上兼容，通过 `ToolDefinitionWrapper` 桥接转换。

---

## 三层接口体系

### 1. Tool（AI 层基础接口）

位置：`packages/ai/src/types.ts:327-331`

```typescript
export interface Tool<TParameters extends TSchema = TSchema> {
    name: string;       // 工具名称，供 LLM 调用时引用
    description: string; // LLM 系统提示词中的工具描述
    parameters: TParameters; // TypeBox JSON Schema，定义参数结构
}
```

这是最简接口，仅保留 LLM 理解工具所需的最小信息。参数使用 [TypeBox](https://github.com/sinclairzx81/typebox) 定义，用于自动生成 JSON Schema。

### 2. AgentTool（Agent 运行时接口）

位置：`packages/agent/src/types.ts:361-384`

```typescript
export interface AgentTool<TParameters extends TSchema = TSchema, TDetails = any> extends Tool<TParameters> {
    label: string;  // 人类可读标签，用于 UI 展示

    /** 可选的参数兼容垫片，在 schema 校验前转换原始参数 */
    prepareArguments?: (args: unknown) => Static<TParameters>;

    /** 执行函数 */
    execute: (
        toolCallId: string,                          // 本次调用的唯一 ID
        params: Static<TParameters>,                 // 已通过校验的参数
        signal?: AbortSignal,                        // 取消信号
        onUpdate?: AgentToolUpdateCallback<TDetails>, // 流式更新回调
    ) => Promise<AgentToolResult<TDetails>>;

    /** 执行模式：sequential=串行, parallel=并行 */
    executionMode?: ToolExecutionMode;
}
```

核心要点：
- `execute()` 返回 `AgentToolResult<TDetails>`，包含发给 LLM 的 `content` 和供 UI/日志使用的结构化 `details`
- `onUpdate` 回调支持流式部分结果（如 bash 命令的实时输出）
- `toolCallId` 用于关联 LLM 的 tool call 与结果

### 3. ToolDefinition（扩展层完整定义）

位置：`packages/coding-agent/src/core/extensions/types.ts:426-473`

```typescript
export interface ToolDefinition<TParams extends TSchema, TDetails = unknown, TState = any> {
    name: string;
    label: string;
    description: string;
    promptSnippet?: string;         // 系统提示词中的"可用工具"片段
    promptGuidelines?: string[];    // 追加到提示词"指南"部分的规则
    parameters: TParams;
    renderShell?: "default" | "self"; // TUI 渲染模式
    prepareArguments?: (args: unknown) => Static<TParams>;
    executionMode?: ToolExecutionMode;

    execute(
        toolCallId: string,
        params: Static<TParams>,
        signal: AbortSignal | undefined,
        onUpdate: AgentToolUpdateCallback<TDetails> | undefined,
        ctx: ExtensionContext,  // ← 比 AgentTool 多一个上下文参数
    ): Promise<AgentToolResult<TDetails>>;

    renderCall?: (args, theme, context) => Component;       // 自定义调用渲染
    renderResult?: (result, options, theme, context) => Component; // 自定义结果渲染
}
```

相比 `AgentTool` 增加了：
- **提示词集成**: `promptSnippet` 和 `promptGuidelines` 自动注入系统提示
- **TUI 渲染**: `renderCall()` 和 `renderResult()` 控制终端 UI 展示
- **渲染外壳**: `renderShell` 决定使用标准彩色 shell 框架还是工具自控渲染
- **扩展上下文**: `ctx: ExtensionContext` 提供 CWD、配置等资源

### 工具结果结构

```typescript
// 工具返回结果 (packages/agent/src/types.ts:345-355)
export interface AgentToolResult<T> {
    content: (TextContent | ImageContent)[];  // 发送给 LLM 的内容
    details: T;                                // 结构化详情（UI/日志用）
    terminate?: boolean;                       // 提示是否终止本轮批次
}

// 发给 LLM 的消息 (packages/ai/src/types.ts:292-300)
export interface ToolResultMessage<TDetails = any> {
    role: "toolResult";
    toolCallId: string;
    toolName: string;
    content: (TextContent | ImageContent)[];
    details?: TDetails;
    isError: boolean;
    timestamp: number;
}
```

### 桥接转换

`ToolDefinitionWrapper`（`packages/coding-agent/src/core/tools/tool-definition-wrapper.ts`）提供双向适配：

```typescript
// ToolDefinition → AgentTool（扩展工具注册时使用）
function wrapToolDefinition(definition, ctxFactory?): AgentTool

// AgentTool → ToolDefinition（用户提供纯运行时覆盖时使用）
function createToolDefinitionFromAgentTool(tool): ToolDefinition
```

---

## 内置工具

Pi 内置 7 个工具，分为两组：

| 工具 | 参数 | 文件 | 说明 |
|------|------|------|------|
| **read** | `path`, `offset?`, `limit?` | `read.ts` | 读取文件内容，支持文本和图片 |
| **bash** | `command`, `timeout?` | `bash.ts` | 执行 Shell 命令，流式输出 |
| **edit** | `path`, `edits: [{oldText, newText}]` | `edit.ts` | 精准文件编辑，支持多处修改 |
| **write** | `path`, `content` | `write.ts` | 写入文件，自动创建目录 |
| **grep** | `pattern`, `path?`, `glob?`, `ignoreCase?`, `literal?`, `context?`, `limit?` | `grep.ts` | 使用 ripgrep 搜索文件内容 |
| **find** | `pattern`, `path?`, `limit?` | `find.ts` | 使用 fd 按 glob 搜索文件 |
| **ls** | `path?`, `limit?` | `ls.ts` | 列出目录内容 |

### 工具分组策略

```
Coding Mode (编程模式):  read, bash, edit, write
ReadOnly Mode (只读模式): read, grep, find, ls
All Tools (全部):         上述 7 个
```

工厂函数位于 `packages/coding-agent/src/core/tools/index.ts`：

- `createCodingTools()` / `createCodingToolDefinitions()` — 编程模式工具集
- `createReadOnlyTools()` / `createReadOnlyToolDefinitions()` — 只读模式工具集
- `createAllTools()` / `createAllToolDefinitions()` — 全部工具集

### 统一工具创建模式

每个内置工具遵循相同的构造范式：

```
定义 TypeBox Schema
    → 定义 Operations 接口（可插拔实现，支持远程委托如 SSH）
    → 定义 ToolDetails 类型（截断元数据、文件路径等）
    → createXxxToolDefinition(cwd, options?) → ToolDefinition
    → wrapToolDefinition() → AgentTool
    → createXxxTool(cwd, options?) 快捷方式
```

`Operations` 接口的设计允许将具体操作委托给不同后端。例如 `BashOperations` 定义了 spawn 子进程的能力，`createLocalBashOperations()` 提供本地实现，未来可以创建 `createSshBashOperations()` 实现远程执行。

### 各工具详解

#### Read 工具

- **功能**: 读取文件内容，支持文本和图片（图像文件自动识别）
- **偏移/限制**: 通过 `offset` 和 `limit` 参数支持分页读取大文件
- **自动截断**: 默认最多 2000 行或 50KB，使用 `truncateHead()` 保留文件头部
- **Operations 接口**: `ReadOperations` 定义了 `readFile()`, `access()`, `detectImageMimeType()` 方法
- **图片处理**: 见下方 [Read 工具图片处理](#read-工具图片处理) 专节

---

##### Read 工具图片处理

**判断流程**：通过 `ReadOperations.detectImageMimeType()` 检测文件 MIME 类型（默认实现为 `detectSupportedImageMimeTypeFromFile`，位于 `src/utils/mime.ts`），返回非空值即判定为图片。

**处理流程**（[read.ts:249-277](packages/coding-agent/src/core/tools/read.ts#L249-L277)）：

```
检测 MIME 类型 → 是图片
    ↓
读取文件为 Buffer → 转 base64 字符串
    ↓
是否启用 autoResizeImages（默认 true）？
    ├─ 是 → resizeImage() 压缩至 2000x2000 以下
    │         ├─ 成功 → 返回压缩后的 base64 + mimeType
    │         └─ 失败 → 返回纯文本提示 "could not be resized"
    └─ 否 → 直接返回原始 base64 + mimeType
    ↓
检查当前模型是否支持图片（getNonVisionImageNote）
    └─ 不支持 → 附加文本说明 "Current model does not support images"
```

**返回数据结构**：成功读取图片时，`content` 数组包含两个并列元素：

```typescript
content = [
    // 元素1: 文本说明
    { type: "text", text: "Read image file [image/jpeg]\n[Resized to 1200x800]" },
    // 元素2: 图片数据
    { type: "image", data: "base64...", mimeType: "image/jpeg" },
];
```

`ImageContent` 类型定义在 [packages/ai/src/types.ts:240-244](packages/ai/src/types.ts#L240-L244)：

```typescript
export interface ImageContent {
    type: "image";
    data: string;     // base64 编码的图片数据
    mimeType: string; // 如 "image/jpeg"、"image/png"
}
```

若 `autoResizeImages` 关闭，文本说明中省略尺寸信息。图片与说明文本作为独立的 content 元素并列放在数组中，由下游 LLM API 层负责将 base64 编码到多模态请求中。

#### Bash 工具

- **功能**: 执行 Shell 命令
- **流式输出**: 通过 `onUpdate` 回调实时推送 stdout/stderr 片段，带限流
- **超时控制**: `timeout` 参数控制命令超时时间
- **尾部截断**: 使用 `truncateTail()` 保留命令输出的最后部分（通常是错误信息和最终结果）
- **Spawn 钩子**: 支持 `BashSpawnHook` 在子进程创建前进行拦截
- **Operations 接口**: `BashSpawnContext` 封装子进程的 spawn 和流读取

#### Edit 工具

- **功能**: 精确文本替换，支持多处不连续修改（`edits` 数组）
- **变异队列**: 同一文件的操作通过 `withFileMutationQueue()` 串行化，不同文件并行执行
- **验证**: 查找 `oldText` 精确匹配，不存在时报错
- **Operations 接口**: `EditOperations` 定义 `readFile()` 和 `writeFile()` 方法

#### Write 工具

- **功能**: 写入文件内容，自动创建父目录
- **变异队列**: 与 edit 工具共享相同的队列机制
- **Operations 接口**: `WriteOperations` 定义 `mkdir()` 和 `writeFile()` 方法

#### Grep 工具

- **功能**: 基于 ripgrep 的内容搜索
- **尊重 .gitignore**: 自动跳过被忽略的文件
- **丰富过滤**: 支持正则/字面量、路径限定、glob 过滤、忽略大小写、上下文行数
- **行截断**: 每行匹配结果使用 `truncateLine()` 截断至 500 字符
- **Operations 接口**: `GrepOperations` 定义 `exec()` 方法执行 ripgrep 命令

#### Find 工具

- **功能**: 基于 fd 的文件名搜索
- **尊重 .gitignore**: 自动跳过被忽略的文件和目录
- **Operations 接口**: `FindOperations` 定义 `exec()` 方法执行 fd 命令

#### Ls 工具

- **功能**: 列出目录内容，按字母排序
- **Operations 接口**: `LsOperations` 定义 `readdir()` 方法

---

## 工具执行管线

工具执行由 `AgentLoop.executeToolCalls()`（`packages/agent/src/agent-loop.ts:373-718`）驱动。

### 完整流程

```
LLM 返回 AssistantMessage
        │
        ▼
┌── executeToolCalls() ──────────────────────────────┐
│ 1. 检查是否有 executionMode="sequential" 的工具      │
│ 2. 是 → executeToolCallsSequential()               │
│    否 → executeToolCallsParallel()                  │
└─────────────────────────────────────────────────────┘
        │
        ▼
┌── prepareToolCall() ───────────────────────────────┐
│ 1. 从 context.tools 中按名称查找工具                 │
│    - 找不到 → 返回 "Tool X not found" 错误          │
│ 2. 调用 tool.prepareArguments() 兼容垫片            │
│ 3. validateToolArguments() 校验 TypeBox schema      │
│    - TypeBox Value.Convert() 类型转换                │
│    - JSON Schema 手动兜底强制                         │
│ 4. beforeToolCall hook → 可阻断执行                  │
└─────────────────────────────────────────────────────┘
        │
        ▼
┌── executePreparedToolCall() ───────────────────────┐
│ 1. 调用 tool.execute(toolCallId, args, signal, cb) │
│ 2. onUpdate 回调发射 tool_execution_update 事件     │
│ 3. 捕获异常 → 转换为错误结果                        │
└─────────────────────────────────────────────────────┘
        │
        ▼
┌── finalizeExecutedToolCall() ──────────────────────┐
│ 1. afterToolCall hook → 可覆盖 content/details     │
│ 2. 字段级合并（非深度合并）                         │
│ 3. 检查 terminate 标志                              │
└─────────────────────────────────────────────────────┘
        │
        ▼
创建 ToolResultMessage → 追加到 context.messages → 进入下一轮
```

### 并行 vs 串行

**并行执行**（默认）：
1. **预检阶段**: 所有 tool call 依次通过 `prepareToolCall()`
2. 如果某个工具被阻断/不存在，立即返回错误
3. **执行阶段**: `Promise.all()` 并发执行所有已预检的工具

**串行执行**：逐个工具依次完成 prepare → execute → finalize 全链路。

当任一工具声明 `executionMode: "sequential"` 时，整个批次切换为串行模式。

### 终止逻辑

```typescript
// 仅当批次中 EVERY 个工具都设置了 terminate=true 时才提前终止
function shouldTerminateToolBatch(finalizedCalls): boolean {
    return finalizedCalls.length > 0 && finalizedCalls.every(f => f.result.terminate === true);
}
```

---

## 截断系统

所有工具的输出都经过统一截断控制，防止上下文膨胀。

位置：`packages/coding-agent/src/core/tools/truncate.ts`

### 默认限制

| 常量 | 值 | 用途 |
|------|-----|------|
| `DEFAULT_MAX_LINES` | 2000 | 最大行数 |
| `DEFAULT_MAX_BYTES` | 50KB | 最大字节数 |
| `GREP_MAX_LINE_LENGTH` | 500 | grep 单行最大字符数 |

两个限制取**先命中者**。

### 截断策略

| 函数 | 策略 | 适用场景 |
|------|------|---------|
| `truncateHead()` | 保留开头 N 行/字节 | read 工具 — 保留文件头部 |
| `truncateTail()` | 保留末尾 N 行/字节 | bash 工具 — 保留最终输出 |
| `truncateLine()` | 单行截断 + `[truncated]` 后缀 | grep 工具 — 匹配行截断 |

### TruncationResult 元数据

```typescript
interface TruncationResult {
    content: string;          // 截断后的内容
    truncated: boolean;       // 是否发生截断
    truncatedBy: "lines" | "bytes" | null; // 哪个限制触发
    totalLines: number;       // 原始总行数
    totalBytes: number;       // 原始总字节数
    outputLines: number;      // 输出的行数
    outputBytes: number;      // 输出的字节数
    lastLinePartial: boolean; // 最后一行是否部分截断
    firstLineExceedsLimit: boolean; // 首行是否超过字节限制
    maxLines: number;         // 应用的行数限制
    maxBytes: number;         // 应用的字节限制
}
```

丰富的元数据使下游可以准确判断截断原因和程度，UI 可据此显示截断提示。

---

## 文件变异队列

`edit` 和 `write` 工具使用共享的文件变异队列机制。

位置：`packages/coding-agent/src/core/tools/file-mutation-queue.ts`

### 设计动机

当 LLM 并发调用 edit/write 修改同一文件时，需要串行化操作以避免竞态条件。

### 实现原理

```typescript
// 按文件的 realpath 维护独立的 Promise 链
const queues: Map<string, Promise<void>> = new Map();

function withFileMutationQueue<T>(
    filePath: string,
    fn: () => Promise<T>,
): Promise<T> {
    const resolvedPath = realpathSync(filePath);
    const previous = queues.get(resolvedPath) ?? Promise.resolve();
    const next = previous.then(fn, fn); // 无论成功失败都串行
    queues.set(resolvedPath, next);
    return next;
}
```

关键特性：
- **同文件串行**: 同一文件的所有操作排队执行
- **不同文件并行**: 不同文件的操作互不影响，可并发
- **错误不阻塞**: `.then(fn, fn)` 确保即使前一个操作失败，后续操作仍会执行

---

## 注册与组合

### AgentSession 中的工具注册

`AgentSession` 维护 4 个 Map 来管理工具（`packages/coding-agent/src/core/agent-session.ts:304-307`）：

```typescript
private _toolRegistry: Map<string, AgentTool> = new Map();           // 运行时执行用
private _toolDefinitions: Map<string, ToolDefinitionEntry> = new Map(); // 扩展定义存储
private _toolPromptSnippets: Map<string, string> = new Map();        // 系统提示词片段
private _toolPromptGuidelines: Map<string, string[]> = new Map();    // 系统提示词指南
```

### 工具来源（3 个渠道）

1. **内置工具**: 通过 `_buildRuntime()` 从 `createAllToolDefinitions()` 构建，或 `baseToolsOverride` 覆盖
2. **SDK 自定义工具**: 通过 `AgentSessionConfig.customTools` 传入 `ToolDefinition[]`
3. **扩展工具**: 通过 `ExtensionAPI.registerTool()` 注册到扩展的 tools map，再经 `wrapRegisteredTools()` 包装

### 注册流程

```
内置工具    ──→ createXxxToolDefinition() → _toolRegistry
                                         ↓
SDK 工具     ──→ createToolDefinitionFromAgentTool() → _toolRegistry
                                         ↓
扩展工具     ──→ ExtensionAPI.registerTool()
                  → wrapRegisteredTools()
                    → wrapToolDefinition(ctxFactory) → _toolRegistry
                                         ↓
                              构建系统提示词（合并 snippets + guidelines）
```

---

## 事件与 Hook

### Agent 级 Hook

位置：`packages/agent/src/types.ts:256-276`

```typescript
// 工具执行前钩 — 可阻断执行
beforeToolCall?: (
    context: BeforeToolCallContext,
    signal?: AbortSignal,
) => Promise<BeforeToolCallResult | undefined>;

// 工具执行后钩 — 可修改结果
afterToolCall?: (
    context: AfterToolCallContext,
    signal?: AbortSignal,
) => Promise<AfterToolCallResult | undefined>;
```

`AgentSession._installAgentToolHooks()` 将这些 hook 连接到扩展系统的事件机制。

### 扩展事件

工具相关的事件类型（`packages/coding-agent/src/core/extensions/types.ts`）：

| 事件类型 | 触发时机 | 用途 |
|---------|---------|------|
| `ToolCallEvent` (按工具名细分) | 执行前 | 可拦截、修改输入参数 |
| `ToolResultEvent` (按工具名细分) | 执行后 | 可修改返回内容 |
| `ToolExecutionStartEvent` | 执行开始 | 通知 UI |
| `ToolExecutionUpdateEvent` | 流式更新 | 推送部分结果 |
| `ToolExecutionEndEvent` | 执行结束 | 通知 UI 完成 |

扩展通过事件订阅拦截：

```typescript
pi.on("tool_call", handler);      // 可阻断或修改输入参数
pi.on("tool_result", handler);    // 可修改结果内容
```

### Agent 生命周期事件

除了工具级事件，`AgentLoop` 还通过 `AgentEventEmitter` 发射全局事件：

```
tool_execution_start    → ToolExecutionStartEvent
tool_execution_update   → ToolExecutionUpdateEvent (流式)
tool_execution_end      → ToolExecutionEndEvent
message_delta           → 流式文本增量
message                 → 完整消息
agent_end               → 会话结束
```

---

## TUI 渲染

工具调用和结果通过 `ToolExecutionComponent` 在终端 UI 中渲染。

位置：`packages/coding-agent/src/modes/interactive/components/tool-execution.ts`

### 渲染器解析

```
扩展注册的 renderCall/renderResult（优先级高）
    ↓ 未找到回退到
内置工具的 renderCall/renderResult
```

### 渲染外壳模式

| `renderShell` | 行为 |
|---|---|
| `"default"` | 标准彩色 shell 框架（绿色命令、灰色输出） |
| `"self"` | 工具自定义渲染，不受标准外壳约束 |

### 渲染状态管理

```typescript
interface ToolRenderContext<TState, TArgs> {
    invalidate(): void;           // 触发重新渲染
    state: TState;               // 工具私有状态
    lastComponent: Component | null; // 上次渲染结果
}
```

支持增量渲染：
- `executionStarted` — 执行开始
- `argsComplete` — 参数渲染完成
- `isPartial` — 流式部分结果
- `expanded` — 用户展开详情
- `isError` — 执行错误

图像渲染支持 Kitty 终端协议，当终端支持时可通过 Kitty graphics protocol 直接显示图片。

---

## 扩展自定义工具

通过 `defineTool()` 辅助函数创建自定义工具：

```typescript
import { defineTool, Type } from "@earendil-works/pi-coding-agent";

const myTool = defineTool({
    name: "my-tool",
    label: "My Tool",
    description: "Description for the LLM",
    promptSnippet: "Use my-tool for custom operations",
    promptGuidelines: ["Always validate input before using"],
    parameters: Type.Object({
        input: Type.String({ description: "Input parameter" }),
    }),
    execute: async (toolCallId, params, signal, onUpdate, ctx) => {
        // 实现工具逻辑
        return {
            content: [{ type: "text", text: `Result: ${params.input}` }],
            details: { /* 结构化详情 */ },
        };
    },
    renderCall: (args, theme, context) => {
        // 返回 TUI Component
        return new Text(`my-tool: ${args.input}`, { color: "cyan" });
    },
    renderResult: (result, options, theme, context) => {
        return new Box(result.content.map(c => c.text).join(""));
    },
});

// 通过 AgentSessionConfig.customTools 传入
const session = new AgentSession({
    customTools: [myTool],
    // ...
});
```

### 关键约束

- `parameters` 必须使用 TypeBox (`Type.Object`, `Type.String` 等) 定义
- `execute` 返回的 `content` 数组元素为 `TextContent` 或 `ImageContent`
- `renderShell: "self"` 配合自定义 `renderCall` 可完全控制 TUI 展示
- 通过 `ctx.cwd` 访问当前工作目录，通过 `ctx.config` 访问配置
