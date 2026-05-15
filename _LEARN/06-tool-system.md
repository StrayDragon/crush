# 06 - 工具系统分析

## 工具执行流程

[MermaidChart:./diagrams/tool-execution.mmd]

## 工具清单

### 文件操作工具

| 工具名 | 文件 | 功能 | 关键参数 |
|--------|------|------|----------|
| `view` | `view.go` | 读取文件内容 | `file_path`, `offset`, `limit` |
| `edit` | `edit.go` | 精确字符串替换 | `file_path`, `old_string`, `new_string`, `replace_all` |
| `write` | `write.go` | 创建/覆盖文件 | `file_path`, `content` |
| `multiedit` | `multiedit.go` | 批量字符串替换 | `edits[]` (多个编辑操作) |
| `glob` | `glob.go` | 文件模式匹配 | `pattern` |
| `grep` | `grep.go` | 内容搜索 | `pattern`, `path` |

### Shell 执行工具

| 工具名 | 文件 | 功能 | 关键参数 |
|--------|------|------|----------|
| `bash` | `bash.go` | 执行 Shell 命令 | `command`, `description`, `timeout` |
| `job_output` | `bash_job.go` | 获取后台任务输出 | `job_id` |
| `job_kill` | `bash_job.go` | 终止后台任务 | `job_id` |

### 搜索工具

| 工具名 | 文件 | 功能 |
|--------|------|------|
| `grep` | `grep.go` | 正则搜索文件内容 |
| `glob` | `glob.go` | Glob 模式匹配文件名 |
| `ls` | `ls.go` | 目录列表 |

### 开发工具

| 工具名 | 文件 | 功能 |
|--------|------|------|
| `diagnostics` | `diagnostics.go` | LSP 诊断信息 |
| `references` | `references.go` | LSP 代码引用 |
| `lsp_restart` | `lsp_restart.go` | 重启 LSP 服务器 |

### Web 工具

| 工具名 | 文件 | 功能 |
|--------|------|------|
| `fetch` | `fetch.go` | HTTP 请求 |
| `web_search` | `web_search.go` | Web 搜索 |
| `download` | `download.go` | 文件下载 |
| `agentic_fetch` | `agentic_fetch.go` | AI 增强的 Web 获取 |

### MCP 工具

| 工具名 | 文件 | 功能 |
|--------|------|------|
| `list_mcp_resources` | `mcp_list_resources.go` | 列出 MCP 资源 |
| `read_mcp_resource` | `mcp_read_resource.go` | 读取 MCP 资源 |
| `mcp_*` | `mcp_tool.go` | 动态 MCP 工具代理 |

### 系统工具

| 工具名 | 文件 | 功能 |
|--------|------|------|
| `crush_info` | `crush_info.go` | 系统信息 |
| `crush_logs` | `crush_logs.go` | 日志访问 |
| `todos` | `todos.go` | 任务管理 |

## 工具接口

所有工具实现统一的 `AgentTool` 接口：

```go
type AgentTool interface {
    Info() ToolInfo
    Run(ctx context.Context, call ToolCall) (ToolResponse, error)
}

type ToolInfo struct {
    Name        string         // 工具名称
    Description string         // 工具描述
    Parameters  json.RawMessage // JSON Schema 参数定义
}

type ToolResponse struct {
    Content  string          // 输出内容
    IsError  bool            // 是否错误
    Metadata any             // 附加元数据
    StopTurn bool            // 是否终止当前轮次
    Media    *Media          // 媒体内容（图片等）
}
```

## Bash 工具详解

### 安全机制

```go
// 禁用命令列表
var bannedCommands = []string{
    "sudo", "su",              // 提权
    "apt", "pacman", "brew",   // 包管理
    "systemctl", "service",    // 系统服务
    "curl", "wget", "ssh",     // 网络工具
    "npm", "pip", "gem",       // 包安装
}

// 自动后台化阈值
const autoBackgroundThreshold = 60 * time.Second
// 运行超过 60s 的命令自动转为后台任务
```

### 执行流程

```
1. 命令检查 → 是否在禁用列表
2. 权限请求 → Hook 预检 → 用户确认
3. 执行命令 → 启动子进程
4. 监控输出 → 100ms 轮询
5. 超时处理 → 60s 自动后台化
6. 结果格式化 → 行号 + 工作目录 + 退出码
7. 输出截断 → 30KB 上限
```

### 后台任务管理

```
用户选择后台运行
  或命令超过 60s
    → 创建 BackgroundShell
    → 返回 job_id
    → 用户通过 job_output 获取输出
    → 用户通过 job_kill 终止任务
```

## Edit 工具详解

### 文件编辑安全链

```
1. 必须先 view 过文件（filetracker 检查）
2. 检查文件是否被外部修改（ModTime 对比）
3. 创建备份版本（file history）
4. 精确字符串匹配替换
5. LSP 通知文件变更
```

### 编辑操作

```go
type EditParams struct {
    FilePath   string `json:"file_path"`
    OldString  string `json:"old_string"`
    NewString  string `json:"new_string"`
    ReplaceAll bool   `json:"replace_all,omitempty"`
}
```

关键规则：
- `old_string` 必须在文件中**唯一匹配**（除非 `replace_all=true`）
- `old_string` 和 `new_string` 不能相同
- 文件修改后会自动通知 LSP

### 文件历史追踪

```go
type FileHistory interface {
    Record(ctx, sessionID, filePath, oldContent, newContent)
    // 记录每次修改，支持回滚
}
```

## View 工具详解

### 文件查看能力

```
支持的文件类型:
  → 文本文件: 行号显示，支持 offset/limit 分页
  → 图片文件: 检测 MIME 类型，Base64 编码发送
  → Skill 文件: 特殊处理（更大文件限制）

限制:
  → 最大 200KB (skill 文件更大)
  → 必须在 working directory 内
```

### LSP 集成

```
view 文件时:
  → 自动打开 LSP 客户端（如果未启动）
  → 获取诊断信息
  → 注入到工具结果中
```

## MCP 工具代理

### MCP 连接管理

```
配置 MCP 服务器:
{
  "mcp": {
    "my-server": {
      "type": "stdio",          // stdio | http | sse
      "command": "my-mcp-server",
      "args": ["--port", "3000"]
    }
  }
}
```

### 连接生命周期

```
启动 → 连接 MCP 服务器
  → 发现工具和资源列表
  → 工具注册到 Agent 可用列表
  → 监听工具/资源变更事件
  → 断连自动重连
```

### MCP 工具执行

```
LLM 调用 MCP 工具
  → 匹配到 MCP Tool 代理
  → 转发参数到 MCP 服务器
  → 解析返回结果
  → 格式化为 ToolResponse
```

## Hook 系统

### Hook 事件类型

```go
const (
    EventPreToolUse = "pre_tool_use"  // 工具执行前
    EventPostToolUse = "post_tool_use" // 工具执行后（未来）
)
```

### Hook 执行规则

```
多个 Hook 按顺序执行:
  Deny > Allow > None
  Halt 是粘性的（一旦设置不可撤销）
  Input patches 按顺序合并

Hook 可以:
  → 允许工具执行（跳过用户确认）
  → 拒绝工具执行
  → 修改工具输入参数
  → 终止整个对话轮次
```

### Hook 配置示例

```json
{
  "hooks": {
    "pre_tool_use": [
      {
        "matcher": "bash",
        "command": "check-command-safety.sh",
        "timeout": 5000
      }
    ]
  }
}
```

## 权限系统

### 权限请求流

```
工具执行 → permission.Request()
  → Hook 预检 (PreToolUse)
    → Allow → 直接执行（跳过用户确认）
    → Deny → 返回拒绝
    → None → 继续用户确认
  → 用户确认对话框
    → Allow → 执行
    → Deny → 返回拒绝
    → Allow Always → 记住选择，下次自动通过
```

### 安全命令白名单

某些命令自动通过权限检查：
- `ls`, `pwd`, `echo`
- `git status`, `git log`, `git diff`
- `cat`, `head`, `tail` (只读命令)

### 禁用操作

- 禁用命令: `sudo`, `su`, `rm -rf /` 等
- 禁用参数: `npm install -g`, `pip install` 等
- 路径限制: 只能操作 working directory 内的文件
