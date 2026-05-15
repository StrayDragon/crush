# 08 - 开发者指南

## 开发环境搭建

### 前置要求

- Go 1.26.3+
- SQLite (构建时通过 modernc.org/sqlite，无需 CGO)
- Task (任务运行器): `go install github.com/go-task/task/v3/cmd/task@latest`
- golangci-lint: `go install github.com/golangci/golangci-lint/cmd/golangci-lint@latest`

### 可选工具

- Nix: 使用 `flake.nix` 一键搭建开发环境
- Goreleaser: 本地构建发布版本

### 快速开始

```bash
# 克隆项目
git clone <repo-url> crush && cd crush

# 安装依赖
go mod download

# 构建
task build

# 运行
./crush

# 运行测试
task test

# 代码检查
task lint:fix

# 格式化
task fmt
```

## 项目构建

### Taskfile 命令

| 命令 | 用途 |
|------|------|
| `task build` | 构建二进制 |
| `task test` | 运行全部测试 |
| `task lint` | 代码检查 |
| `task lint:fix` | 检查并自动修复 |
| `task fmt` | 格式化代码 (gofumpt) |
| `task modernize` | 代码现代化 |

### 构建约束

```
CGO_ENABLED=0          → 纯 Go，无 C 依赖
GOEXPERIMENT=greenteagc → 实验性 GC 优化
```

### 跨平台构建

使用 Goreleaser 自动构建：
- macOS (amd64, arm64)
- Linux (amd64, arm64)
- Windows (amd64)
- BSD (amd64)

## 代码风格与规范

### 格式化

- 使用 `gofumpt`（比 gofmt 更严格）
- 使用 `goimports` 管理导入顺序
- 缩进: Tab

### 命名规范

- 文件名: `snake_case.go`
- 导出类型: `PascalCase`
- 未导出类型: `camelCase`
- 接口: `PascalCase` + `er` 后缀（如 `SessionAgent`, `FileTracker`）
- 常量: `PascalCase` 或 `UPPER_SNAKE_CASE`

### 错误处理

```go
// 结构化日志
logger.Error("Failed to execute tool",
    "tool", toolName,
    "error", err,
)

// 错误返回
if err != nil {
    return fmt.Errorf("execute tool %s: %w", toolName, err)
}
```

### 注释规范

- 导出函数必须有 doc comment
- 注释以函数名开头
- 不解释 What，解释 Why

## 架构模式

### 1. 依赖注入

服务通过构造函数注入依赖：

```go
func NewCoordinator(
    cfg config.ConfigStore,
    permissions permission.Service,
    lspManager lsp.Manager,
    // ...
) *coordinator
```

### 2. 接口抽象

关键组件通过接口解耦：

```go
type SessionAgent interface { ... }
type FileTracker interface { ... }
type FileHistory interface { ... }
type PermissionService interface { ... }
```

### 3. 事件驱动

通过 PubSub 实现组件间松耦合：

```go
// 发布事件
pubsub.Publish(EventType, payload)

// 订阅事件
pubsub.Subscribe(EventType, handler)
```

### 4. 模板方法

Agent 系统使用模板方法模式：
- 基类定义执行骨架
- 子类实现具体逻辑

## 添加新工具

### 步骤

1. 在 `internal/agent/tools/` 创建新文件

```go
// my_tool.go
package tools

type myTool struct {
    permissions permission.Service
    // 其他依赖
}

func NewMyTool(perms permission.Service) *myTool {
    return &myTool{permissions: perms}
}

func (t *myTool) Info() fantasy.ToolInfo {
    return fantasy.ToolInfo{
        Name:        "my_tool",
        Description: "Does something useful",
        Parameters:  json.RawMessage(`{
            "type": "object",
            "properties": {
                "input": {"type": "string", "description": "The input"}
            },
            "required": ["input"]
        }`),
    }
}

func (t *myTool) Run(ctx context.Context, call fantasy.ToolCall) (fantasy.ToolResponse, error) {
    // 1. 解析参数
    var params struct {
        Input string `json:"input"`
    }
    if err := json.Unmarshal(call.Input, &params); err != nil {
        return fantasy.ToolResponse{}, err
    }

    // 2. 请求权限
    approved, err := t.permissions.Request(ctx, permission.CreatePermissionRequest{
        ToolName: "my_tool",
        Action:   "execute",
    })
    if !approved {
        return fantasy.NewTextErrorResponse("Permission denied"), nil
    }

    // 3. 执行逻辑
    result := doSomething(params.Input)

    // 4. 返回结果
    return fantasy.ToolResponse{
        Content: result,
    }, nil
}
```

2. 在 `coordinator.buildTools()` 中注册

```go
allTools = append(allTools, tools.NewMyTool(c.permissions))
```

3. 在 Agent 配置中添加到 `AllowedTools`

4. 编写测试

```go
func TestMyTool(t *testing.T) {
    // 设置 mock
    tool := NewMyTool(mockPermissions)
    // 测试执行
    resp, err := tool.Run(ctx, call)
    assert.NoError(t, err)
    assert.Contains(t, resp.Content, "expected")
}
```

## 添加新 LLM Provider

### 步骤

1. 在 Fantasy 库中实现 Provider 接口（外部依赖）
2. 在 `internal/config/` 添加 Provider 配置
3. 在配置 Schema 中添加 Provider 类型
4. 测试 Provider 连接和功能

## 添加新 MCP 集成

### 步骤

1. 在 `crush.json` 中配置 MCP 服务器

```json
{
  "mcp": {
    "my-server": {
      "type": "stdio",
      "command": "my-mcp-server",
      "args": ["--option", "value"],
      "env": {"KEY": "value"}
    }
  }
}
```

2. Crush 自动发现和注册 MCP 工具
3. 通过 `allowed_mcp` 控制工具访问

## 测试指南

### 单元测试

```go
func TestEditTool(t *testing.T) {
    // 使用 testify
    assert := assert.New(t)

    // 设置 fixture
    tmpFile := createTempFile(t, "original content")

    // 执行测试
    tool := NewEditTool(...)
    resp, err := tool.Run(ctx, call)

    // 验证结果
    assert.NoError(err)
    assert.False(resp.IsError)
}
```

### Golden Files 测试

UI 组件使用 Golden Files 测试：

```
testdata/
├── expected_output.golden  → 期望的渲染结果
└── input.json              → 测试输入

// 更新 golden files
go test ./... -update
```

### 集成测试

```
testdata/
├── testdata/          → 测试数据目录
├── *_test.go          → 带测试数据的集成测试
```

## 数据库开发

### 使用 sqlc

1. 修改 `schema.sql` 添加表
2. 在 `query.sql` 添加查询
3. 运行 `sqlc generate` 生成 Go 代码
4. 使用生成的代码进行数据库操作

```sql
-- query.sql
-- name: GetSession :one
SELECT * FROM sessions WHERE id = ?;

-- name: CreateSession :one
INSERT INTO sessions (id, workspace) VALUES (?, ?)
RETURNING *;
```

## 调试

### 启用 pprof

```bash
CRUSH_PPROF=1 crush
# 访问 http://localhost:6060/debug/pprof/
```

### 日志查看

```bash
crush logs           # 查看日志
crush logs --follow  # 实时查看
```

### 性能分析

```bash
go test -bench=. -benchmem ./internal/agent/
```
