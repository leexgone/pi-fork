# 会话历史与记忆管理系统 — 技术研究报告

## 1. 概述

Pi 项目的会话历史与记忆管理系统由三个独立但概念一致的子系统组成，分别面向不同运行环境：

| 子系统 | 包 | 存储后端 | 树结构 | 用途 |
|--------|-----|---------|--------|------|
| **Coding Agent** | `coding-agent` | 文件系统 JSONL | 是 | CLI 主会话系统 |
| **Agent Harness** | `agent` | 可插拔（内存/JSONL/自定义） | 是 | SDK 抽象层，供集成方使用 |
| **Web UI** | `web-ui` | IndexedDB | 否（线性） | 浏览器端轻量会话 |

三者共享统一的消息类型体系（`AgentMessage` 联合类型），但存储格式和分支能力有差异。

---

## 2. 核心数据模型

### 2.1 消息类型体系

所有子系统共享 `AgentMessage` 联合类型，通过 TypeScript 声明合并扩展：

**基础类型**（`@earendil-works/pi-agent-core` 定义）：
- `UserMessage` — 用户输入（文本或多模态）
- `AssistantMessage` — 助手响应（含思考过程、工具调用、用量统计）
- `ToolResultMessage` — 工具执行结果（含错误标记、时间戳）

**扩展类型**（`coding-agent` 通过声明合并注入）：
- `BashExecutionMessage` — bash 命令执行记录（含 `excludeFromContext` 标记，对应 `!!` 前缀命令）
- `CustomMessage` — 扩展自定义消息（由扩展通过 `session_before_compact` 等事件注入）
- `BranchSummaryMessage` — 分支导航摘要
- `CompactionSummaryMessage` — 上下文压缩摘要

### 2.2 会话树条目（SessionTreeEntry）

在 Coding Agent 和 Agent Harness 中，消息不以线性数组存储，而是组织为一棵树。每个条目共享以下结构：

```typescript
interface SessionTreeEntryBase {
  type: string;             // 条目类型
  id: string;               // 8 字符短 ID（带碰撞检测）
  parentId: string | null;  // null 表示根节点
  timestamp: string;        // ISO 8601 时间戳
}
```

**完整条目类型清单**：

| 条目类型 | 用途 | 参与 LLM 上下文 |
|----------|------|-----------------|
| `MessageEntry` | 用户/助手/工具结果消息 | 是 |
| `ThinkingLevelChangeEntry` | 思考级别变更 | 否（状态标记） |
| `ModelChangeEntry` | 会话中模型切换 | 否（状态标记） |
| `CompactionEntry` | 压缩摘要 + 保留边界 | 间接（通过摘要） |
| `BranchSummaryEntry` | 分支放弃摘要 | 是 |
| `CustomEntry` | 扩展自定义状态 | 否 |
| `CustomMessageEntry` | 扩展自定义消息 | 是 |
| `LabelEntry` | 用户书签/标记 | 否 |
| `SessionInfoEntry` | 会话显示名称 | 否 |
| `LeafEntry` | 内部叶指针记录 | 否 |

**LeafEntry 的特殊性**：这是一个纯内部条目，不参与上下文构建。当叶指针移动时，系统追加一个 `LeafEntry`（`parentId = oldLeafId`, `targetId = newLeafId`），用于持久化记录叶指针的变迁历史。

### 2.3 树结构示意

```
Root
 └── [user msg] (id=a, parentId=null)
      └── [assistant] (id=b, parentId=a)
           └── [user msg] (id=c, parentId=b)
                ├── [assistant] (id=d, parentId=c) ← ACTIVE LEAF
                └── [user msg] (id=e, parentId=c)  ← alternate branch
                     └── [assistant] (id=f, parentId=e)
```

**关键不变式**：条目是 append-only 的。分支操作不会修改或删除已有条目——只移动叶指针。这保证了完整的对话历史始终可追溯，类似于 Git 的提交图设计。

---

## 3. 存储实现

### 3.1 Coding Agent — 文件系统 JSONL

**存储路径**：`~/.pi/agent/sessions/--<encoded-cwd>--/<timestamp>_<uuid>.jsonl`

工作目录路径通过将 `/` 和 `\` 替换为 `-` 编码为安全目录名，包裹在 `--` 分隔符中。

**文件格式**：JSONL（每行一个独立 JSON 对象）
- 第 1 行：`SessionHeader`（类型 `session`，含版本号、UUID、时间戳、cwd、父会话引用）
- 后续行：`SessionTreeEntry` 条目

**延迟写入策略**：
- 首个助手消息到达前，条目仅缓存在内存中（`flushed = false`）
- 首个助手消息到达时，一次性将所有缓冲条目写入文件
- 之后每条新消息通过 `appendFileSync` 追加
- 这避免了为空会话或中断的对话创建空文件

**自动迁移**：
- v1 → v2：为线性条目添加 `id`/`parentId` 树结构
- v2 → v3：将 `hookMessage` 角色重命名为 `custom`

### 3.2 Agent Harness — 可插拔存储

`SessionStorage` 接口定义了存储操作的抽象层：
- `InMemorySessionStorage` — 内存存储，维护 `byId` Map 和 `SessionTreeEntry[]` 数组
- `JsonlSessionStorage` — 文件 JSONL 存储，格式与 Coding Agent 一致

`SessionRepo` 接口定义了会话仓库操作（CRUD + fork），对应两种实现：
- `InMemorySessionRepo` — 内存仓库
- `JsonlSessionRepo` — 文件仓库

这种分层使 SDK 集成方可以自行实现存储后端（如数据库、云存储）。

### 3.3 Web UI — IndexedDB

最简化的线性存储：
- `sessions` 对象存储：完整会话数据
- `sessions-metadata` 对象存储：轻量元数据（用于快速列表）

`SessionData` 是扁平结构：`{ id, title, model, thinkingLevel, messages[], createdAt, lastModified }`。

**无树结构**：Web UI 不支持会话分支，消息以线性数组存储。这是因为 Web UI 面向简单聊天场景，不需要 CLI 的分支探索能力。

---

## 4. 会话分支管理

Pi 的会话树支持三种分支操作，各有不同语义：

### 4.1 `/tree` — 树内导航

在**同一个会话文件**内移动叶指针：
1. `branch(branchFromId)` 设置 `leafId = branchFromId`
2. 后续 `appendMessage()` 会作为该条目的子节点追加
3. 离开旧分支时触发分支摘要生成
4. 被放弃的分支仍保留在文件中，只是不在活跃路径上

**设计要点**：叶指针移动是可逆操作，历史不丢失。

### 4.2 `/fork` — 从特定点创建新会话

创建一个**新的会话文件**，仅包含从根到分叉点的路径：
- 新会话头部包含 `parentSession` 字段指向原始文件
- 两种分叉模式：
  - `"before"`（默认）：分叉点在选中用户消息的父节点（排除该消息）
  - `"at"`：分叉点在选中条目本身（包含它）
- 从非用户消息以 `"before"` 模式分叉会报错

**与 Git 的类比**：类似于从历史提交创建新分支。

### 4.3 `/clone` — 提取活跃分支

`createBranchedSession(leafId)` 创建仅包含从根到当前叶路径的新会话：
- 标签（labels）被保留并作为新条目重建
- 新会话的 `parentSession` 指向原始文件

---

## 5. 分支摘要系统

当 `/tree` 导航离开一个分支时，系统会生成该分支的摘要：

### 5.1 流程

```
collectEntriesForBranchSummary(oldLeafId, targetId)
  → 构建旧叶和新目标到根的路径集合
  → 找到最深公共祖先
  → 收集从旧叶到公共祖先的所有条目（不含祖先本身）

prepareBranchEntries(entries, tokenBudget)
  → 从新到旧遍历条目
  → 提取消息（跳过工具结果、设置条目）
  → 累积嵌套分支摘要中的文件操作记录
  → 尊重 token 预算限制

generateBranchSummary(options)
  → 向 LLM 发送结构化提示词
  → 返回摘要文本 + 读取/修改文件列表
  → 文件列表追加到摘要，存储在 BranchSummaryEntry.details 中
```

### 5.2 提示词差异

分支摘要使用不同的引导语：
> "The user explored a different conversation branch before returning here."

结构化格式与压缩摘要类似但更简洁：Goal / Constraints / Progress / Key Decisions / Next Steps。

---

## 6. 上下文压缩系统

### 6.1 触发机制

两个触发场景：

**溢出触发**（`isContextOverflow`）：
- LLM 返回上下文溢出错误
- 仅重试一次（`_overflowRecoveryAttempted` 防护）
- 移除错误消息后自动压缩并重试

**阈值触发**（`shouldCompact`）：
```typescript
contextTokens > contextWindow - reserveTokens
```
默认配置下，128K 模型在 `contextTokens > 111616` 时触发。

### 6.2 Token 计数策略

混合计数策略兼顾精度与性能：

1. **精确计数**：使用最后一次成功的助手消息的 `usage` 数据
2. **启发式估算**：`chars / 4`（保守高估）
   - 图片估算为 4800 字符（1200 tokens）
   - 工具调用使用 `name + JSON 参数` 字符数

```
总 token 数 = 最后成功助手的 usage.totalTokens + 其后消息的 chars/4 估算
```

### 6.3 切割点算法

`findCutPoint()` 的核心逻辑：
1. 从最新消息向后遍历，累加 token 估算值
2. 当累加值达到 `keepRecentTokens`（默认 20000）时停止
3. 找到最近的有效切割点：
   - **有效切割点**：user/assistant/custom/bashExecution
   - **不可切割**：toolResult（工具结果必须与调用配对）
4. 如果切割在 assistant 消息处，向前找到 turn 起点处理 split-turn

### 6.4 摘要生成

**防对话陷阱**：对话先序列化为纯文本格式，防止模型将历史当作正在进行的对话：
```
[User]: text
[Assistant]: response
[Assistant thinking]: reasoning
[Assistant tool calls]: read(path="foo.ts")
[Tool result]: output (截断 2000 字符)
```

**双模式摘要**：
- **首次摘要**：创建结构化摘要（Goal/Constraints/Progress/Key Decisions/Next Steps/Critical Context）
- **更新摘要**：将新信息合并到已有摘要中，保留旧数据

**Split-turn 处理**：当单个对话轮次超出预算时，并行生成两个摘要后合并：
1. 历史摘要（之前的完整轮次）
2. 当前轮次前缀摘要（更短的 token 预算）

### 6.5 文件操作追踪

压缩和分支摘要都会追踪 `readFiles` 和 `modifiedFiles`，从工具调用中提取路径。这些信息以 XML 格式追加到摘要中，跨多次压缩累积：
```xml
<read-files>path/to/file.ts</read-files>
<modified-files>path/to/file.ts</modified-files>
```

---

## 7. 上下文重建

压缩后，`buildSessionContext()` 从叶到根遍历树，构建发送给 LLM 的消息列表：

```
1. 从 leafId 沿 parentId 链回溯到根，收集路径条目
2. 提取当前 thinkingLevel 和 model（最后出现者生效）
3. 找到路径上最近的 CompactionEntry
4. 如果存在压缩：
   a. 先发射 CompactionSummaryMessage
   b. 包含 firstKeptEntryId 到 compaction 之间的保留消息
   c. 再包含 compaction 之后的所有消息
5. 将 BranchSummaryEntry 和 CustomMessageEntry 转换为消息等价物
6. 返回 { messages, thinkingLevel, model }
```

关键设计：压缩不会丢弃任何数据——历史条目仍保留在文件中，只是 `buildSessionContext` 选择性地跳过它们。

---

## 8. 消息转换管线

在每次 LLM 调用前，`convertToLlm()` 将 `AgentMessage[]` 转换为 LLM 可接受的格式：

| 输入类型 | 转换输出 |
|----------|---------|
| bashExecution（排除） | 过滤掉（`!!` 前缀命令） |
| bashExecution（保留） | → User 消息：`Ran \`cmd\`\n\`\`\`output\`\`\`` |
| custom | → User 消息 |
| branchSummary | → User 消息：`<summary>` XML 包裹 |
| compactionSummary | → User 消息：`<summary>` XML 包裹 |
| user/assistant/toolResult | 原样传递 |

扩展的 `context` 事件可在 `convertToLlm` 之前拦截并修改消息列表。

---

## 9. 架构设计分析

### 9.1 核心设计原则

1. **Append-only 历史**：所有条目只追加不删除，类似 Git 提交日志。分支操作只移动指针，不修改已有数据。

2. **存储与逻辑分离**：Agent Harness 通过 `SessionStorage`/`SessionRepo` 接口抽象，使存储后端可替换。Coding Agent 使用文件系统实现，Web UI 使用 IndexedDB。

3. **压缩非破坏性**：压缩操作在树中标记边界（`CompactionEntry`），但历史条目仍保留。切换压缩策略或重新加载时可以从头重建上下文。

4. **混合 Token 计数**：利用提供商返回的精确 usage 数据作为锚点，仅对新增内容使用 `chars/4` 启发式估算，在精度和性能间取得平衡。

### 9.2 与 Git 的类比

| Git 概念 | Pi 会话系统 |
|----------|-------------|
| 提交（commit） | SessionTreeEntry |
| 父提交（parent） | `parentId` 指针 |
| HEAD | `leafId` 叶指针 |
| 分支（branch） | 不同的叶路径 |
| `git branch` | `/tree` 导航 |
| `git branch <new>` | `/fork` 创建新会话 |
| `git cherry-pick` | `/clone` 提取路径 |
| 提交消息 | 分支/压缩摘要 |

### 9.3 局限性

1. **无语义记忆**：没有向量数据库或语义记忆系统。"记忆"完全通过会话树和压缩摘要实现。

2. **Web UI 无分支**：Web UI 使用线性消息数组，不支持会话树分支。

3. **JSONL 全文加载**：会话文件在加载时全量读入内存，大会话可能存在性能问题。

4. **Token 估算精度**：`chars/4` 是保守高估，对代码密集型消息可能显著偏差。

---

## 10. 关键文件清单

| 文件路径 | 职责 |
|----------|------|
| `packages/coding-agent/src/core/session-manager.ts` | 主会话管理器：条目类型、文件 I/O、树遍历、分支操作 |
| `packages/coding-agent/src/core/agent-session.ts` | 运行时编排：会话切换、压缩触发、溢出恢复 |
| `packages/coding-agent/src/core/agent-session-runtime.ts` | 会话生命周期管理：fork、clone、import |
| `packages/coding-agent/src/core/compaction/compaction.ts` | 核心压缩逻辑：切割点检测、摘要生成 |
| `packages/coding-agent/src/core/compaction/branch-summarization.ts` | 分支摘要收集 |
| `packages/coding-agent/src/core/compaction/utils.ts` | 共享工具：文件追踪、序列化、摘要提示词 |
| `packages/coding-agent/src/core/messages.ts` | 消息类型声明合并、convertToLlm 转换管线 |
| `packages/agent/src/harness/types.ts` | 核心类型定义：SessionTreeEntry、SessionStorage、SessionRepo |
| `packages/agent/src/harness/session/session.ts` | Session 门面类 |
| `packages/agent/src/harness/session/jsonl-storage.ts` | 文件 JSONL 存储 |
| `packages/agent/src/harness/session/memory-storage.ts` | 内存存储 |
| `packages/ai/src/utils/overflow.ts` | 跨提供商溢出错误检测（22+ 正则模式） |
| `packages/web-ui/src/storage/stores/sessions-store.ts` | Web UI IndexedDB 存储 |
