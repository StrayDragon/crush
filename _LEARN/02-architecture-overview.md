# 02 - 架构概览

## 系统架构图

[MermaidChart:./diagrams/architecture.mmd]

## 分层架构

Crush 采用清晰的分层架构，从上到下依次为：

### 1. CLI 层 (`internal/cmd`)
**职责**: 命令行接口，参数解析，子命令分发

```
root.go        → 主命令入口，全局 flag
run.go         → 非交互式执行（crush run "prompt"）
session.go     → 会话管理
dirs.go        → 目录管理
models.go      → 模型选择
projects.go    → 项目管理
logs.go        → 日志查看
schema.go      → Schema 操作
```

技术选型: **Cobra** 命令行框架

### 2. TUI 层 (`internal/ui`)
**职责**: 终端用户界面，交互反馈

```
model/ui.go          → 主 UI Model（Bubble Tea）
model/chat.go        → 聊天消息列表
model/keys.go        → 键盘快捷键映射
styles/styles.go     → Lipgloss 样式系统
common/markdown.go   → Markdown 渲染（Glamour）
```

技术选型: **Bubble Tea v2** + **Lipgloss v2** + **Glamour v2**

### 3. 应用层 (`internal/app`)
**职责**: 服务编排，生命周期管理

- 串联 CLI → Agent → DB → LSP → UI 各层
- 管理应用启动/关闭流程
- 协调 Agent 运行

### 4. Agent 层 (`internal/agent`)
**职责**: AI Agent 核心逻辑

```
coordinator.go       → Agent 编排器（核心中的核心）
agent.go             → SessionAgent 实现
templates/           → 系统提示词模板（Go template）
```

### 5. 工具层 (`internal/agent/tools`)
**职责**: 可执行工具集合

```
bash.go              → Shell 命令执行
edit.go              → 文件编辑
view.go              → 文件查看
write.go             → 文件写入
multiedit.go         → 批量编辑
grep.go              → 内容搜索
glob.go              → 文件匹配
ls.go                → 目录列表
fetch.go             → HTTP 请求
web_search.go        → Web 搜索
download.go          → 文件下载
diagnostics.go       → LSP 诊断
references.go        → LSP 引用
mcp_*.go             → MCP 工具
```

### 6. 基础设施层

| 模块 | 路径 | 职责 |
|------|------|------|
| 数据库 | `internal/db` | SQLite 持久化（sqlc 生成） |
| LSP | `internal/lsp` | 语言服务器协议集成 |
| 配置 | `internal/config` | 配置加载与管理 |
| Hooks | `internal/hooks` | 工具执行钩子 |
| Session | `internal/session` | 会话状态管理 |
| Skills | `internal/skills` | Agent 能力扩展 |
| Shell | `internal/shell` | 命令执行引擎 |
| Event | `internal/event` | 遥测（PostHog） |
| PubSub | `internal/pubsub` | 内部消息传递 |
| OAuth | `internal/oauth` | 认证（Copilot 支持） |
| Proto | `internal/proto` | 协议定义 |

## Agent 交互时序

[MermaidChart:./diagrams/agent-flow.mmd]

核心交互流程:
1. 用户输入 → TUI 层捕获 → App 层转发 → Coordinator
2. Coordinator 构建上下文（系统提示 + 历史 + 工具列表）→ 发送到 LLM
3. LLM 返回流式响应（文本/工具调用）
4. 工具调用 → 权限检查 → 执行 → 结果回传 LLM
5. 循环直到 LLM 不再请求工具调用

## 数据流管道

[MermaidChart:./diagrams/data-flow.mmd]

## 关键设计决策

### 1. Go Template 用于 Prompt 模板
```
templates/coder.md.tpl    → 主编程 Agent 系统提示
templates/task.md.tpl     → 任务分析 Agent 系统提示
templates/summary.md      → 会话摘要模板
templates/title.md        → 标题生成模板
```
模板支持变量注入（工作目录、环境信息等），便于动态生成提示。

### 2. sqlc 代码生成用于数据库
避免手写 SQL 和 ORM，通过 sqlc 从 SQL 查询生成类型安全的 Go 代码。

### 3. Fantasy 作为 LLM 抽象层
Charm 的 Fantasy 库提供统一的 LLM 调用接口，屏蔽不同 Provider 的 API 差异。

### 4. Bubble Tea v2 的 Elm Architecture
```
Model (状态) + Update (消息处理) + View (渲染)
```
清晰的单向数据流，易于理解和测试。

### 5. PubSub 事件驱动
组件间通过 PubSub 解耦，支持：
- Agent 状态变更通知
- MCP 连接状态更新
- 文件变更事件

## 依赖关系图

```
cmd → app → agent → tools
              ↓        ↓
            config    lsp
              ↓        ↓
             db ←─────┘
              ↓
           session

ui → app (间接通过 channel/callback)
hooks ← config
event ← app
pubsub ← app
```

## 配置优先级

```
1. 命令行参数 (最高)
2. .crush.json (项目级)
3. crush.json (项目级)
4. ~/.config/crush/crush.json (全局)
5. 默认值 (最低)
```
