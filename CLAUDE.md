# CLAUDE.md

此文件为 Claude Code (claude.ai/code) 在此代码库中工作时提供指导。

## 项目概述

**pi** 是一个 TypeScript 单体仓库，构建 AI 编程代理框架。它提供可扩展的编程代理，包含终端 UI、统一多提供商 LLM API、代理运行时和 Web UI 组件。发布在 npm 组织 `@earendil-works` 下。需要 Node >= 20。

## 单体仓库结构

5 个包采用锁步版本管理，按依赖顺序构建：

| 包 | 说明 |
|---------|-------------|
| `packages/tui` | 终端 UI 库（差异渲染、模糊搜索、编辑器组件、键绑定）。无内部依赖。使用 `node --test` 测试。 |
| `packages/ai` | 多提供商 LLM API（OpenAI、Anthropic、Google、Mistral、Bedrock、Azure）。自动生成模型列表。使用 vitest 测试。 |
| `packages/agent` | 代理运行时（工具调用、传输抽象、状态管理）。依赖 `ai`。使用 vitest 测试。 |
| `packages/coding-agent` | 交互式编程代理 CLI（`pi` 二进制）。依赖 `ai`、`agent`、`tui`。使用 vitest 测试。 |
| `packages/web-ui` | AI 聊天 Web 组件（mini-lit/lit + Tailwind）。依赖 `ai`、`tui`。 |

## 常用命令

```bash
npm install                          # 安装依赖
npm run check                        # Biome 代码检查/格式化 + tsgo 类型检查（每次代码变更后运行）
./test.sh                            # 运行测试（无 API Key 时跳过依赖 LLM 的测试）
npm run release:patch                # 发布 Bug 修复/新功能
npm run release:minor                # 发布 API 破坏性变更
```

**单个包测试：** `npx tsx ../../node_modules/vitest/dist/cli.js --run test/specific.test.ts`（需在包根目录下运行）

**切勿运行：** `npm run dev`、`npm run build`、`npm test`

## 代码规范

- 大幅修改前需完整阅读文件，不要仅依赖搜索结果
- 禁止使用 `any` 类型（除非绝对必要）
- 禁止内联导入（`await import(...)`、`import("pkg").Type`）——仅使用标准顶层导入
- 仅被调用一次的单行辅助函数应内联化
- 查阅 `node_modules` 确认外部 API 类型定义，不要猜测
- 切勿为修复类型错误而降级或移除代码——升级依赖即可
- 切勿硬编码键检查——所有键绑定必须可配置
- 切勿直接修改 `packages/ai/src/models.generated.ts`——应更新 `packages/ai/scripts/generate-models.ts`

## 代码检查与格式化

使用 Biome v2.3.5：制表符缩进，行宽 120。代码变更后运行 `npm run check`——修复所有错误、警告和信息提示后再提交。

## 变更日志

每个包有独立的 `CHANGELOG.md`。新条目添加到 `## [Unreleased]` 下的对应子分类（`### Added`、`### Fixed`、`### Changed`、`### Breaking Changes`、`### Removed`）。切勿修改已发布版本的内容。格式：`Fixed foo bar ([#123](https://github.com/earendil-works/pi-mono/issues/123))`

## Git 安全

多个代理可能同时工作。**禁止使用** `git add -A`、`git add .`、`git reset --hard`、`git checkout .`、`git clean -fd`、`git stash`、`git commit --no-verify`。始终仅暂存你修改的具体文件。提交信息中包含 `fixes #<号码>` 以关联相关 Issue。

## 关键文件

- `AGENTS.md` — 全面的开发规则（代理的首要参考文档）
- `CONTRIBUTING.md` — 贡献流程与质量标准
- `.pi/` — 配置目录（Git 模板、npm 扩展、提示词模板）
- `packages/coding-agent/examples/extensions/` — 扩展系统示例

## 架构要点

- **提供商抽象层**（`packages/ai`）：统一的 LLM 提供商流式 API。模型列表由 `scripts/generate-models.ts` 自动生成，非手动编辑。
- **扩展系统**（`packages/coding-agent`）：从 `.pi/extensions/` 加载的扩展可添加自定义提供商、技能和模式。
- **会话格式**：编程代理使用基于会话的工作流，支持内容压缩。

## 研究文档

- `docs/architecture.md` — 项目整体架构分析与设计思路
- `docs/tool-system.md` — 工具系统详细设计：三层接口、7 个内置工具、执行管线、截断系统、事件机制、TUI 渲染与扩展方式

## 工作目的

- 研究该开源项目的设计架构与思路
- 了解智能体开发的技术细节
- 相关研究文档存储于`docs`目录中
- 相关研究文档在存储后，需要同时登记到该文档中
