# 03 - 源码导航指南

## 目录结构全景

```
crush/
├── main.go                          # 程序入口
├── go.mod / go.sum                  # Go 模块定义
├── schema.json                      # 配置 JSON Schema
├── crush.json                       # 本地项目配置
├── AGENTS.md                        # Agent 上下文说明
├── Taskfile.yaml                    # 任务运行器（替代 Makefile）
├── .goreleaser.yml                  # 跨平台发布配置
├── flake.nix                        # Nix 开发环境
├── .golangci.yml                    # Lint 规则
│
├── internal/                        # 核心实现（不可外部导入）
│   ├── app/                         # 应用编排层
│   ├── agent/                       # Agent 核心
│   │   ├── coordinator.go           #   Agent 编排器
│   │   ├── agent.go                 #   SessionAgent 实现
│   │   ├── tools/                   #   36+ 内置工具
│   │   └── templates/               #   系统提示词模板
│   ├── cmd/                         # CLI 命令
│   ├── ui/                          # TUI 实现
│   │   ├── model/                   #   Bubble Tea Models
│   │   ├── styles/                  #   Lipgloss 样式
│   │   └── common/                  #   共享工具
│   ├── config/                      # 配置管理
│   ├── db/                          # SQLite 数据层
│   ├── lsp/                         # 语言服务器集成
│   ├── session/                     # 会话管理
│   ├── hooks/                       # Hook 系统
│   ├── skills/                      # Skills 发现
│   ├── shell/                       # Shell 执行
│   ├── event/                       # 遥测
│   ├── pubsub/                      # 消息总线
│   ├── oauth/                       # OAuth 认证
│   ├── workspace/                   # 工作区管理
│   ├── history/                     # 历史管理
│   ├── proto/                       # 协议定义
│   ├── client/                      # RPC 客户端
│   └── server/                      # RPC 服务端
│
├── docs/                            # 文档
├── .github/                         # CI/CD
├── .agents/skills/                  # 内置 Agent Skills
└── scripts/                         # 构建/开发脚本
```

## 入口分析

### main.go
```go
// 简洁的入口，可选启动 pprof 性能分析
func main() {
    if os.Getenv("CRUSH_PPROF") != "" {
        go http.ListenAndServe("localhost:6060", nil)
    }
    cmd.Execute()
}
```

### cmd/root.go
- 使用 Cobra 定义根命令和全局 flag
- 初始化 Viper 配置
- 设置日志级别
- 根据子命令分发到对应 handler

## 核心模块详解

### 1. internal/agent/coordinator.go — 系统心脏

Coordinator 是整个系统的编排核心，职责包括：

```
┌─────────────────────────────────────┐
│           Coordinator                │
│                                      │
│  buildTools()     → 构建可用工具集    │
│  buildAgentModels() → 选择大/小模型   │
│  buildSystemPrompt() → 组装系统提示   │
│  Run()            → 执行 Agent 调用   │
│  buildMessages()  → 组装消息历史      │
│                                      │
│  依赖:                              │
│  - config (模型/工具配置)            │
│  - permissions (权限服务)            │
│  - lspManager (LSP 管理)            │
│  - history (文件编辑历史)            │
│  - filetracker (文件追踪)            │
└─────────────────────────────────────┘
```

**关键方法调用链**:
```
Run()
  → buildAgentModels()   // 获取大/小模型配置
  → buildSystemPrompt()  // 渲染系统提示模板
  → buildTools()         // 注册所有可用工具
  → agent.Stream()       // 流式调用 LLM
    → OnTextDelta()      // 文本增量回调
    → OnToolCall()       // 工具调用回调
    → OnToolResult()     // 工具结果回调
    → OnReasoningStart() // 思考过程开始
    → OnReasoningEnd()   // 思考过程结束
```

### 2. internal/agent/agent.go — SessionAgent 实现

```go
type SessionAgent interface {
    Run(ctx, call SessionAgentCall) → (SessionAgentResult, error)
    SetModels(large, small Model)
    SetTools(tools []AgentTool)
    SetSystemPrompt(prompt string)
    Cancel()
}
```

SessionAgent 维护一次完整的 LLM 对话循环，处理：
- 消息队列（并发请求排队）
- 上下文窗口管理（自动摘要）
- 流式响应处理
- 工具调用循环

### 3. internal/agent/tools/ — 工具集

每个工具遵循统一接口：
```go
type AgentTool interface {
    Info() ToolInfo           // 名称、描述、参数 Schema
    Run(ctx, call) → Response // 执行工具逻辑
}
```

工具注册在 `coordinator.buildTools()` 中完成，按 Agent 配置过滤。

### 4. internal/ui/model/ui.go — TUI 核心

Bubble Tea 的主 Model，管理全局状态：

```go
type UI struct {
    state      uiState       // 当前状态机位置
    chat       Chat          // 聊天组件
    textarea   Textarea      // 输入组件
    dialog     Dialog        // 对话框组件
    completions Completions  // 自动补全
    // ...
}
```

状态机: `uiOnboarding → uiInitialize → uiChat / uiLanding`

### 5. internal/config/ — 配置管理

```go
type ConfigStore struct {
    // 支持多级配置覆盖
    // 全局 ~/.config/crush/crush.json
    // 项目 ./crush.json 或 .crush.json
}

type Config struct {
    Models    map[SelectedModelType]SelectedModel  // 大/小模型
    Providers map[string]ProviderConfig            // LLM Provider
    MCP       map[string]MCPConfig                 // MCP 服务器
    Hooks     map[HookEvent][]Hook                 // 钩子
    LSP       map[string]LSPConfig                 // 语言服务器
    Options   Options                              // 全局选项
}
```

### 6. internal/db/ — 数据持久化

使用 sqlc 从 SQL 生成类型安全的 Go 代码：
- `schema.sql` — 表结构定义
- `query.sql` — 查询定义
- `*.go` — 生成的 Go 代码（不要手动编辑）

实体: Sessions, Messages, Files

### 7. internal/lsp/ — 语言服务器集成

```go
type Manager struct {
    clients map[string]*Client  // 按语言维护 LSP 客户端
}

// 懒初始化: 首次访问文件时才启动对应 LSP
// 支持: diagnostics, references, hover 等
```

## 测试结构

```
internal/
├── agent/tools/*_test.go        → 工具单元测试
├── config/*_test.go             → 配置加载测试
├── ui/model/*_test.go           → UI 组件测试（golden files）
├── db/*_test.go                 → 数据库测试
├── session/*_test.go            → 会话管理测试
└── testdata/                    → 测试数据
```

测试框架: 标准 `testing` + `testify`

## 构建与开发

```bash
# 构建
task build

# 测试
task test

# Lint
task lint:fix

# 格式化
task fmt

# 代码现代化
task modernize
```

构建约束:
- `CGO_ENABLED=0` — 纯 Go，无 C 依赖
- `GOEXPERIMENT=greenteagc` — 实验性 GC

## 代码风格

- `gofumpt` 格式化（比 gofmt 更严格）
- `goimports` 管理导入
- 结构化日志（大写开头）
- 语义化 commit 前缀
