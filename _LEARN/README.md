# Crush - 全面学习指引

> Crush 是由 [Charm](https://charm.land) 团队打造的终端原生 AI 编程助手，基于 Go + Bubble Tea v2 构建。

## 项目定位

Crush 是一个运行在终端中的 AI 编程助手，它将 LLM 能力与开发工具深度集成：
- **多 LLM 支持**: Anthropic, OpenAI, Gemini, Bedrock, Azure, OpenRouter 等
- **终端原生**: 基于 Bubble Tea v2 的 TUI 框架
- **可扩展**: MCP (Model Context Protocol) 服务器 + Agent Skills + Hooks
- **跨平台**: macOS, Linux, Windows, BSD

## 学习路线

### 第一阶段：理解产品
| 文档 | 内容 | 视角 |
|------|------|------|
| [01-product-perspective.md](./01-product-perspective.md) | 产品价值、竞品对比、核心场景 | 产品视角 |
| [09-user-guide.md](./09-user-guide.md) | 功能矩阵、使用场景、工作流 | 用户视角 |

### 第二阶段：理解架构
| 文档 | 内容 | 视角 |
|------|------|------|
| [02-architecture-overview.md](./02-architecture-overview.md) | 系统架构、模块划分、依赖关系 | 架构视角 |
| [05-ui-design-analysis.md](./05-ui-design-analysis.md) | TUI 设计、组件体系、交互设计 | 设计视角 |

### 第三阶段：深入源码
| 文档 | 内容 | 视角 |
|------|------|------|
| [03-source-code-guide.md](./03-source-code-guide.md) | 目录结构、入口分析、模块导航 | 开发者视角 |
| [04-core-logic-deep-dive.md](./04-core-logic-deep-dive.md) | Agent 循环、上下文管理、流式处理 | 开发者视角 |
| [06-tool-system.md](./06-tool-system.md) | 工具体系、权限系统、MCP 集成 | 开发者视角 |
| [07-agent-prompt-engineering.md](./07-agent-prompt-engineering.md) | Prompt 模板、Agent 类型、模型策略 | AI 工程视角 |

### 第四阶段：实践
| 文档 | 内容 | 视角 |
|------|------|------|
| [08-developer-guide.md](./08-developer-guide.md) | 构建、测试、贡献指南 | 开发者视角 |
| [10-config-reference.md](./10-config-reference.md) | 配置体系、Provider 设置、Hook 配置 | 运维视角 |

## 图解索引

| 图表 | 描述 |
|------|------|
| [architecture.mmd](./diagrams/architecture.mmd) | 系统整体架构图 |
| [agent-flow.mmd](./diagrams/agent-flow.mmd) | Agent 交互时序图 |
| [tool-execution.mmd](./diagrams/tool-execution.mmd) | 工具执行流程图 |
| [ui-components.mmd](./diagrams/ui-components.mmd) | UI 组件结构图 |
| [data-flow.mmd](./diagrams/data-flow.mmd) | 数据流处理管道图 |

## 技术栈速览

| 层面 | 技术 |
|------|------|
| 语言 | Go 1.26.3 |
| TUI 框架 | Bubble Tea v2 + Lipgloss v2 |
| LLM 抽象 | Charm Fantasy (多 Provider) |
| 数据库 | SQLite (sqlc 代码生成) |
| Markdown 渲染 | Glamour v2 + Chroma |
| 配置 | YAML/JSON + JSON Schema 验证 |
| 构建 | Taskfile + Goreleaser |
| 代码质量 | golangci-lint + gofumpt + goimports |
