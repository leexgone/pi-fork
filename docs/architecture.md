# Pi 智能体框架 - 架构深度解析

## 一、框架概述

**pi** 是一个 TypeScript 单体仓库构建的 AI 编程代理框架。它提供可扩展的编程代理、终端 UI、统一多提供商 LLM API、代理运行时和 Web UI 组件。

### 整体架构分层

```
┌──────────────────────────────────────────────────────┐
│                    用户交互层                          │
│  InteractiveMode / PrintMode / JsonMode / RpcMode     │
├──────────────────────────────────────────────────────┤
│                    终端 UI 层 (tui)                    │
│  差异渲染 · 模糊搜索 · 编辑器 · 键绑定 · 覆盖层         │
├──────────────────────────────────────────────────────┤
│                  编程代理层 (coding-agent)              │
│  工具系统 · 会话管理 · 扩展系统 · 压缩 · 模型解析        │
├──────────────────────────────────────────────────────┤
│                  代理运行时层 (agent)                   │
│  代理循环 · 工具调用 · 状态管理 · 传输抽象              │
├──────────────────────────────────────────────────────┤
│                  LLM API 层 (ai)                       │
│  提供商抽象 · 事件流 · 消息转换 · 模型注册              │
└──────────────────────────────────────────────────────┘
```

**依赖方向**：`tui` → `ai` → `agent` → `coding-agent`（`web-ui` 依赖 `ai` + `tui`）

---

## 二、包结构详解

### 2.1 `packages/ai` — 统一多提供商 LLM API

#### 核心设计：API 与提供商的解耦

框架将 **API**（底层通信协议）和 **提供商**（商业实体）分开：

- **9 种底层 API**：`openai-completions`、`openai-responses`、`openai-codex-responses`、`azure-openai-responses`、`anthropic-messages`、`bedrock-converse-stream`、`google-generative-ai`、`google-vertex`、`mistral-conversations`
- **28+ 提供商**：Anthropic、OpenAI、Google、Mistral、AWS Bedrock、DeepSeek、Groq、Cerebras、xAI 等

多个提供商可以共用同一 API 实现（如 Groq、Cerebras、xAI 都使用 `openai-completions` API），实现一套代码服务多个提供商。

#### 事件流架构

所有流式响应统一为 `AssistantMessageEventStream`，基于 `EventStream<T, R>` 泛型类：

```
生产者端：push(event)          → 推送事件
消费者端：for await...of       → 实时消费
结果获取：await stream.result() → 等待最终完整消息
```

**11 种事件类型**：
- 生命周期：`start`、`done`、`error`
- 文本块：`text_start`、`text_delta`、`text_end`
- 思考块：`thinking_start`、`thinking_delta`、`thinking_end`
- 工具调用：`toolcall_start`、`toolcall_delta`、`toolcall_end`

每个事件携带 `contentIndex`（内容块位置）和 `partial`（累积的助手消息），不同内容块的事件**可能交错**，消费者必须使用 `contentIndex` 关联事件。

#### 懒加载模式

提供商通过 `register-builtins.ts` 使用动态 `import()` 懒加载。只有实际使用某提供商时才加载其 SDK 依赖，避免不必要的内存占用和加载时间。

#### 跨提供商消息转换

`transformMessages()` 处理不同提供商间的消息格式转换：
- 思考块：同提供商保留签名，跨提供商转为纯文本
- 工具调用 ID：自动规范化（如 OpenAI Responses 的超长 ID 转换为 Anthropic 可接受的格式）
- 图片降级：非视觉模型自动替换图片为占位文本
- 孤立工具调用：自动生成错误工具结果

#### 对外 API

| 函数 | 说明 |
|------|------|
| `stream(model, context, options)` | 完整流式调用，支持提供商特定选项 |
| `complete(model, context, options)` | 流式调用的便捷包装，返回最终结果 |
| `streamSimple(model, context, options)` | 简化的统一接口，内置 thinking/reasoning 映射 |
| `getModel(provider, modelId)` | 类型安全的模型查找 |
| `calculateCost(model, usage)` | 自动计算费用 |

---

### 2.2 `packages/agent` — 代理运行时

#### 两层代理架构

**第一层：底层 `agentLoop` / `agentLoopContinue`**（无状态异步生成器）

```
外层循环：处理 follow-up 消息（代理本该停止但还有新工作）
  └── 内层循环：处理工具调用和 steering 消息
       ├── 注入 steering 消息
       ├── 调用 LLM 获取助手响应
       ├── 执行工具（顺序或并行）
       ├── 调用 prepareNextTurn 钩子
       └── 检查是否应停止
```

**第二层：高层 `Agent` 类**（有状态封装）

- 拥有可变对话转录记录
- 管理生命周期（中止、等待空闲）
- 维护 steering 和 follow-up 队列
- 数组赋值时自动浅拷贝，防止意外共享

#### 工具调用四阶段管线

```
阶段 1：准备 (prepareToolCall)
  ├── 查找工具
  ├── 参数转换 (prepareArguments)
  ├── TypeBox 模式验证
  └── beforeToolCall 钩子（可阻止执行）

阶段 2：执行 (executePreparedToolCall)
  └── 调用 tool.execute()，支持 onUpdate 流式更新

阶段 3：最终化 (finalizeExecutedToolCall)
  └── afterToolCall 钩子（可修改结果内容）

阶段 4：事件发射
  └── 发射 tool_execution_end + ToolResultMessage
```

**执行模式**：支持顺序和并行。并行模式下，先顺序预检（验证 + beforeToolCall），通过的工具通过 `Promise.all` 并发执行。如果批次中有任一工具的 `executionMode` 为 `sequential`，整个批次退回到顺序执行。

#### 扩展消息类型

通过 TypeScript 声明合并（declaration merging）扩展消息类型：

```typescript
// 框架内置消息
type AgentMessage = UserMessage | AssistantMessage | ToolResultMessage

// 应用可通过声明合并添加自定义消息
interface CustomAgentMessages {
    bashExecution: BashExecutionMessage;
    branchSummary: BranchSummaryMessage;
    compactionSummary: CompactionSummaryMessage;
}
```

#### 传输抽象

`StreamFn` 类型抽象了 LLM 调用方式：
- 默认：直接调用提供商 API（`streamSimple`）
- 代理模式：通过后端服务器路由（`streamProxy`），减少带宽消耗

支持动态 API 密钥解析，可在每轮调用前刷新短期令牌（如 GitHub Copilot OAuth）。

---

### 2.3 `packages/coding-agent` — 编程代理 CLI

#### CLI 架构

```
cli.ts → main.ts
  ├── 1. 离线模式检查
  ├── 2. 包管理命令 (install/remove/update/list/config)
  ├── 3. 参数解析 (args.ts)
  ├── 4. 模式解析 (interactive / print / json / rpc)
  ├── 5. 会话创建 (continue/resume/session/fork/new)
  ├── 6. 运行时工厂 (settings + model registry + extensions)
  ├── 7. 模型解析
  └── 8. 模式分发
```

**四种运行模式**：
- **interactive**：完整 TUI 交互界面
- **print**：处理后退出，一次性输出
- **json**：JSONL 事件流输出到 stdout
- **rpc**：stdin/stdout JSON-RPC 模式

#### 工具系统

| 工具 | 说明 | 默认启用 |
|------|------|----------|
| `read` | 读取文件内容，支持图片 | 是 |
| `bash` | 执行 Shell 命令，流式输出 | 是 |
| `edit` | 精确查找替换，支持多处并行编辑 | 是 |
| `write` | 创建/覆盖文件，自动创建父目录 | 是 |
| `grep` | 搜索文件内容 | 否 |
| `find` | 按模式查找文件 | 否 |
| `ls` | 列出目录内容 | 否 |

**关键设计**：
- 每个工具接受 `operations` 接口注入，扩展可覆盖文件 I/O 或命令执行（如 SSH 远程执行）
- `edit` 工具使用 `file-mutation-queue` 序列化对同一文件的写入，防止并行工具调用的竞态条件
- `bash` 工具支持进程树追踪和分离子进程管理

#### 会话管理

会话以 **树形结构** 存储为 **JSONL 文件**：

```
SessionHeader (id, timestamp, cwd)
  ├── UserMessage (parentId → Header)
  │   └── AssistantMessage (parentId → UserMessage)
  │       ├── ToolCall 结果...
  │       └── 分支点
  │            ├── 分支 A: 后续对话 → 叶节点
  │            └── 分支 B: 另一条路径
  └── 压缩记录 (summary + firstKeptEntryId)
```

- **追加写入**：首次写入延迟到第一个助手响应（避免空会话产生文件）
- **树形导航**：`leafId` 指针追踪当前位置，分支导航移动指针
- **版本迁移**：当前版本 3，支持从 v1 到 v3 的自动迁移

#### 上下文压缩

当上下文接近模型窗口限制时自动触发：

```
触发条件：contextTokens > contextWindow - reserveTokens
         或提供商返回上下文溢出错误

算法流程：
1. 从最新消息向后遍历，累积估算 token 数（字符数/4 启发式）
2. 保留最近的 keepRecentTokens 不压缩
3. 找到合法切割点（不在工具结果中间）
4. 调用 LLM 生成摘要（结构化：目标、约束、进展、决策、下一步）
5. 插入压缩摘要，替换旧消息
```

**文件操作追踪**：跨压缩保留已读/已修改文件列表，追加到摘要中。

#### 扩展系统

扩展生命周期三阶段：

```
1. 加载阶段
   ├── 项目本地: cwd/.pi/extensions/
   ├── 全局: ~/.pi/agent/extensions/
   ├── 命令行: --extension/-e
   └── 包清单: package.json pi.extensions 字段

2. 绑定阶段
   └── 替换加载时的占位桩函数为真实实现

3. 运行阶段
   └── 事件驱动：20+ 事件类型，覆盖会话/代理/消息/工具/提供商生命周期
```

**扩展能力**：
- 注册自定义工具（带 TypeBox 模式定义）
- 注册斜杠命令
- 注册快捷键
- 注册自定义 CLI 标志
- 注册消息渲染器
- 注册提供商（含 OAuth 支持）
- 会话操作（发送消息、切换模型、设置思考级别等）
- UI 控制（选择器、确认框、输入框、自定义组件等）

---

### 2.4 `packages/tui` — 终端 UI 库

#### 核心抽象

**Component 接口**（函数式渲染）：

```typescript
interface Component {
  render(width: number): string[];   // 纯函数：宽度 → 行数组
  handleInput?(data: string): void;  // 键盘输入处理
  wantsKeyRelease?: boolean;         // 是否需要键释放事件
  invalidate(): void;                // 清除缓存渲染状态
}
```

无虚拟 DOM，无组件级 diffing。diffing 在字符串输出层面完成。

#### 差异渲染

三种渲染策略：

| 策略 | 触发条件 | 操作 |
|------|----------|------|
| 首次渲染 | `previousLines.length === 0` | 直接输出所有行 |
| 全量重绘 | 终端宽度变化 / 内容在视口上方 | 清屏后重绘 |
| 差异更新 | 常规情况 | 仅更新变化的行 |

**差异更新算法**：
1. 逐行比较新旧输出，找到第一个和最后一个变化行
2. 移动光标到 `firstChanged` 行
3. 视口不足时滚动（`\r\n`）
4. 对每个变化行：清除（`\x1b[2K`）+ 输出新内容
5. 所有输出包裹在同步输出协议（`\x1b[?2026h` ... `\x1b[?2026l`）中，消除闪烁

**覆盖层系统**：在基础渲染之上叠加组件（模态框、弹出菜单），每个覆盖层计算其位置和大小后逐行逐列合成到基础行中，覆盖层的变化也参与 diff。

**渲染调度**：`requestRender()` 使用 `process.nextTick` 即时调度（强制模式）或防抖到最小 16ms 间隔（60fps 上限）。

#### 编辑器组件

多行编辑器，76KB 代码量，核心特性：

- **逻辑行 vs 视觉行**：按换行符存储，但渲染时自动换行。光标导航跨视觉行工作
- **智能换行**：使用 `Intl.Segmenter` 实现字素感知的换行
- **粘贴处理**：括号粘贴模式下，超过 10 行的粘贴创建不可见标记，视为原子段
- **Kill ring**：Emacs 风格的剪切/粘贴环
- **撤销栈**：`UndoStack<EditorState>` 使用 `structuredClone()` 深度快照
- **自动补全**：`/` 触发斜杠命令补全，`Tab` 触发文件路径补全
- **跳转模式**：`Ctrl+]` 进入字符跳转

#### 键绑定系统

```
原始终端数据 → StdinBuffer（转义序列组装）
                → TUI.handleInput
                  → inputListeners 链
                    → focusedComponent.handleInput
                      → matchesKey() / kb.matches()
```

- **Kitty 键盘协议**：支持结构化键事件（修饰键、事件类型、备用键）
- **回退模式**：xterm modifyOtherKeys 模式 2
- **命名动作系统**：物理键与语义动作解耦，支持用户自定义覆盖

#### 模糊搜索

评分模型（分数越低匹配越好）：

| 因素 | 分数影响 |
|------|----------|
| 连续匹配 | `-5 × 连续数` |
| 匹配间隔 | `+2 × 间隔` |
| 词边界匹配 | `-10` |
| 位置惩罚 | `+i × 0.1` |
| 完全匹配 | `-100` |
| 字母数字交换 | `+5` |

---

## 三、整体数据流

```
用户输入 (编辑器/快捷键/命令)
    │
    ▼
InteractiveMode (TUI 前端)
    │
    ▼
AgentSession (核心编排)
    ├── Agent (代理循环)
    │     │
    │     ├── 构建上下文 (系统提示 + 消息 + 工具)
    │     │     └── transformMessages() (跨提供商转换)
    │     │
    │     ├── streamSimple() → 调用 LLM
    │     │     └── 提供商适配器 → 发射事件流
    │     │
    │     └── 工具执行管线
    │           └── 内置工具 / 扩展工具
    │
    ├── SessionManager (会话存储)
    │     └── JSONL 树形持久化
    │
    ├── ExtensionRunner (事件分发)
    │     └── 20+ 事件类型钩子
    │
    └── Compaction (上下文压缩)
          └── 自动/手动触发摘要
```

---

## 四、关键设计模式总结

| 模式 | 应用位置 | 说明 |
|------|----------|------|
| **注册表** | `ai` 包的 API 注册、模型注册 | API 名称 → 实现映射，提供商+模型ID → 元数据 |
| **事件流/推拉模型** | `ai` 包的 `EventStream` | `push()` 生产者 + `AsyncIterator` 消费者 + `result()` Promise |
| **区分联合类型** | `AssistantMessageEvent` | 11 种变体，通过 `type` 字段区分 |
| **适配器模式** | 每个提供商实现 | 将原生 API 适配到统一事件流协议 |
| **工厂模式** | 懒加载流函数 | `createLazyStream` 创建懒包装的流函数 |
| **外观模式** | `stream()` / `complete()` | 解析提供商并委托，隐藏底层复杂性 |
| **四阶段管线** | 工具调用 | 准备 → 执行 → 最终化 → 事件发射，每阶段可拦截 |
| **赋值拷贝状态** | `Agent` 状态管理 | 数组赋值时自动拷贝，防止意外共享 |
| **函数式渲染** | TUI 组件 | 纯函数 `(width) => string[]`，渲染时无副作用 |
| **字符串级 diff** | 差异渲染 | 比较渲染后的字符串数组，而非组件树 |
| **声明合并** | 自定义消息类型 | TypeScript 接口扩展，无需修改核心类型 |
| **树形会话存储** | JSONL 文件 | 追加写入，树形结构，支持分支导航和压缩 |

---

## 五、扩展点一览

| 扩展方式 | 实现位置 | 说明 |
|----------|----------|------|
| 自定义工具 | 扩展系统 | 注册带 TypeBox 模式的工具 |
| 斜杠命令 | 扩展系统 | 注册 `/command` 命令 |
| 快捷键 | 扩展系统 | 注册自定义按键绑定 |
| 新提供商 | `ai` 包 | 注册 API + 提供商，更新模型生成脚本 |
| 自定义编辑器 | TUI 接口 | 实现 `EditorComponent` 接口 |
| 新主题 | 扩展系统 | 加载到 `~/.pi/agent/themes/` |
| 技能 (Skills) | 技能系统 | `SKILL.md` 文件，YAML 前置元数据 + Markdown 内容 |
| 提示模板 | 模板系统 | 预定义的提示词模板 |
| 自定义压缩 | 扩展钩子 | `session_before_compact` 事件拦截 |
| SSH 远程执行 | 扩展操作 | 覆盖工具的 `operations` 接口 |
