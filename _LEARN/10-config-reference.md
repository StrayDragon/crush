# 10 - 配置参考手册

## 配置文件位置

### 配置优先级（从高到低）

```
1. 命令行参数
2. .crush.json           (项目级，隐藏文件)
3. crush.json            (项目级)
4. ~/.config/crush/crush.json  (全局)
5. 内置默认值
```

## 配置结构

### 完整配置示例

```json
{
  "providers": {
    "anthropic": {
      "type": "anthropic",
      "api_key": "sk-ant-xxx"
    },
    "openai": {
      "type": "openai",
      "api_key": "sk-xxx"
    },
    "custom": {
      "type": "openai",
      "base_url": "https://api.custom.com/v1",
      "api_key": "key-xxx"
    }
  },

  "models": {
    "large": {
      "provider": "anthropic",
      "model": "claude-opus-4-7"
    },
    "small": {
      "provider": "anthropic",
      "model": "claude-haiku-4-5-20251001"
    }
  },

  "agents": {
    "coder": {
      "allowed_tools": ["bash", "edit", "view", "write", "grep", "glob"],
      "allowed_mcp": ["database", "github"]
    },
    "task": {
      "allowed_tools": ["view", "grep", "glob", "ls"]
    }
  },

  "mcp": {
    "database": {
      "type": "stdio",
      "command": "mcp-server-sqlite",
      "args": ["--db", "myapp.db"],
      "env": {}
    },
    "github": {
      "type": "http",
      "url": "http://localhost:3000/mcp"
    }
  },

  "lsp": {
    "go": {
      "command": "gopls"
    },
    "python": {
      "command": "pyright-langserver",
      "args": ["--stdio"]
    },
    "typescript": {
      "command": "typescript-language-server",
      "args": ["--stdio"]
    }
  },

  "hooks": {
    "pre_tool_use": [
      {
        "matcher": "bash",
        "command": "scripts/check-safety.sh",
        "timeout": 5000
      },
      {
        "matcher": "edit",
        "command": "scripts/notify-edit.sh",
        "timeout": 3000
      }
    ]
  },

  "options": {
    "attribution": true,
    "theme": "auto"
  }
}
```

## Provider 配置

### Anthropic

```json
{
  "type": "anthropic",
  "api_key": "sk-ant-xxx",
  "base_url": "https://api.anthropic.com"  // 可选，默认官方 API
}
```

### OpenAI

```json
{
  "type": "openai",
  "api_key": "sk-xxx",
  "base_url": "https://api.openai.com/v1"  // 可选，默认官方 API
}
```

### Google (Gemini)

```json
{
  "type": "google",
  "api_key": "AIza..."
}
```

### AWS Bedrock

```json
{
  "type": "bedrock",
  "region": "us-east-1",
  "access_key": "AKIA...",
  "secret_key": "..."
}
```

### Azure OpenAI

```json
{
  "type": "azure",
  "base_url": "https://xxx.openai.azure.com",
  "api_key": "xxx",
  "deployment": "my-deployment"
}
```

### OpenRouter

```json
{
  "type": "openai",
  "base_url": "https://openrouter.ai/api/v1",
  "api_key": "sk-or-xxx"
}
```

### 自定义 OpenAI 兼容

```json
{
  "type": "openai",
  "base_url": "https://your-api.com/v1",
  "api_key": "your-key"
}
```

## 模型配置

### 基本模型选择

```json
{
  "models": {
    "large": {
      "provider": "anthropic",
      "model": "claude-opus-4-7",
      "temperature": 0.7,
      "top_p": 0.95,
      "max_tokens": 4096
    },
    "small": {
      "provider": "anthropic",
      "model": "claude-haiku-4-5-20251001"
    }
  }
}
```

### 模型参数

| 参数 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `provider` | string | 必填 | Provider 名称 |
| `model` | string | 必填 | 模型 ID |
| `temperature` | float | Provider 默认 | 采样温度 |
| `top_p` | float | Provider 默认 | 核采样参数 |
| `max_tokens` | int | Provider 默认 | 最大输出 Token |
| `base_url` | string | Provider 默认 | API 地址覆盖 |

## MCP 配置

### STDIO 类型

```json
{
  "type": "stdio",
  "command": "mcp-server-command",
  "args": ["--flag", "value"],
  "env": {
    "ENV_VAR": "value"
  }
}
```

### HTTP 类型

```json
{
  "type": "http",
  "url": "http://localhost:3000/mcp",
  "headers": {
    "Authorization": "Bearer xxx"
  }
}
```

### SSE 类型

```json
{
  "type": "sse",
  "url": "http://localhost:3000/sse"
}
```

### MCP 工具过滤

```json
{
  "agents": {
    "coder": {
      "allowed_mcp": ["database"]  // 只允许 database MCP 的工具
    }
  }
}
```

## LSP 配置

### 基本配置

```json
{
  "lsp": {
    "go": {
      "command": "gopls",
      "args": []
    }
  }
}
```

### 常用语言服务器

| 语言 | command | args |
|------|---------|------|
| Go | `gopls` | `[]` |
| Python | `pyright-langserver` | `["--stdio"]` |
| TypeScript | `typescript-language-server` | `["--stdio"]` |
| Rust | `rust-analyzer` | `[]` |
| C/C++ | `clangd` | `[]` |

## Hook 配置

### PreToolUse Hook

```json
{
  "hooks": {
    "pre_tool_use": [
      {
        "matcher": "bash",
        "command": "scripts/check-bash-safety.sh",
        "timeout": 5000
      }
    ]
  }
}
```

### Hook 字段

| 字段 | 类型 | 说明 |
|------|------|------|
| `matcher` | string | 匹配的工具名（支持 glob） |
| `command` | string | 要执行的 Shell 命令 |
| `timeout` | int | 超时时间（毫秒） |

### Hook 行为

Hook 通过 stdout 控制行为：
- `allow` → 允许工具执行
- `deny` → 拒绝工具执行
- `halt` → 终止整个轮次
- JSON patch → 修改工具参数

## Agent 配置

### 工具白名单

```json
{
  "agents": {
    "coder": {
      "allowed_tools": [
        "bash", "edit", "view", "write",
        "multiedit", "grep", "glob", "ls",
        "fetch", "web_search", "download",
        "diagnostics", "references",
        "list_mcp_resources", "read_mcp_resource",
        "todos"
      ]
    }
  }
}
```

### 可用工具名

| 工具名 | 类别 |
|--------|------|
| `bash` | Shell |
| `edit` | 文件编辑 |
| `view` | 文件查看 |
| `write` | 文件创建 |
| `multiedit` | 批量编辑 |
| `grep` | 搜索 |
| `glob` | 文件匹配 |
| `ls` | 目录列表 |
| `fetch` | HTTP |
| `web_search` | 搜索 |
| `download` | 下载 |
| `diagnostics` | LSP |
| `references` | LSP |
| `lsp_restart` | LSP |
| `list_mcp_resources` | MCP |
| `read_mcp_resource` | MCP |
| `todos` | 任务 |
| `crush_info` | 系统 |
| `crush_logs` | 系统 |

## 环境变量

| 变量 | 说明 |
|------|------|
| `CRUSH_PPROF` | 启用 pprof 性能分析 |
| `CRUSH_LOG_LEVEL` | 日志级别 (debug/info/warn/error) |

## JSON Schema

配置的完整 JSON Schema 定义在项目根目录的 `schema.json` 中，可用于：
- IDE 自动补全
- 配置验证
- 文档生成
