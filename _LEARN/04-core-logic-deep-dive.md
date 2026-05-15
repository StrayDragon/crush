# 04 - 核心逻辑深度分析

## Agent 循环 — 系统的核心引擎

Agent 循环是 Crush 最核心的机制，它驱动了所有 AI 交互。

### 循环概览

```
用户输入 → 构建上下文 → 调用 LLM → 处理响应
                                        ↓
                              ┌─── 是工具调用? ───┐
                              │                    │
                           是 ↓                 否 ↓
                        执行工具              返回文本
                              │                    │
                              └─── 结果回传 LLM ───┘
                                   (继续循环)
```

### 源码路径: `internal/agent/agent.go`

核心流程在 `SessionAgent.Run()` 方法中：

1. **请求入队**: 并发请求通过 channel 排队，保证同一会话串行执行
2. **上下文构建**: 系统提示 + 对话历史 + 工具列表
3. **流式调用**: 通过 Fantasy Agent.Stream() 发起 LLM 请求
4. **增量处理**: 逐块处理文本/工具调用/思考内容
5. **自动摘要**: 上下文窗口不足时触发
6. **结果聚合**: 收集所有工具调用结果，返回最终响应

### 上下文管理策略

```go
// 关键阈值
const (
    largeContextWindowThreshold = 200_000  // 大模型阈值
    largeContextWindowBuffer    = 20_000   // 大模型缓冲
    smallContextWindowRatio     = 0.2      // 小模型比例
)
```

**自动摘要流程**:
```
检测: 剩余 Token < 阈值
  → 暂停当前请求
  → 发送历史到摘要 Agent（使用 summary.md 模板）
  → 生成结构化摘要（状态/文件/策略/待办）
  → 替换历史消息为摘要消息
  → 继续处理排队的请求
```

摘要内容包含:
- **Current state**: 当前进度和状态
- **Files changed**: 已修改文件列表
- **Technical context**: 技术决策和上下文
- **Strategy**: 下一步策略
- **Actionable items**: 具体可执行项

## 消息系统

### 消息类型 (`internal/message/`)

```go
type MessageRole string
const (
    Assistant MessageRole = "assistant"
    User      MessageRole = "user"
    System    MessageRole = "system"
    Tool      MessageRole = "tool"
)
```

### 内容类型

| 类型 | 用途 | 说明 |
|------|------|------|
| TextContent | 文本响应 | 流式增量文本 |
| ToolCall | 工具调用请求 | 工具名 + JSON 参数 |
| ToolResult | 工具执行结果 | 文本/图片/媒体 |
| ReasoningContent | 推理过程 | 可折叠的思考内容 |
| ImageURLContent | 图片附件 | URL 或 Base64 |
| Finish | 完成标记 | 停止原因 |

### 消息构建流程

```
1. 用户消息:
   prompt text + attachments → UserMessage

2. 系统消息:
   template渲染 + workingDir + envInfo → SystemMessage

3. 助手消息:
   textDeltas + toolCalls + reasoning → AssistantMessage (流式构建)

4. 工具消息:
   toolResult + metadata → ToolMessage
```

## 双模型架构

### 大模型 vs 小模型

```
┌──────────────┐     ┌──────────────┐
│  Large Model │     │  Small Model │
│              │     │              │
│ • 复杂推理    │     │ • 快速响应    │
│ • 代码生成    │     │ • 简单任务    │
│ • 长上下文    │     │ • 成本优化    │
│ • 主要 Agent  │     │ • 摘要生成    │
└──────────────┘     └──────────────┘
```

模型选择逻辑 (`coordinator.buildAgentModels()`):
- 从配置获取大/小模型设置
- 支持按 Agent 类型覆盖模型
- 子 Agent 可使用不同模型配置

### Provider 适配

Fantasy 库屏蔽了不同 Provider 的 API 差异：

```
Fantasy Agent
  ├── Anthropic Provider  → Claude 系列
  ├── OpenAI Provider     → GPT 系列
  ├── Google Provider     → Gemini 系列
  ├── Bedrock Provider    → AWS Bedrock
  ├── Azure Provider      → Azure OpenAI
  └── OpenRouter Provider → 多模型路由
```

## 流式响应处理

### 回调机制

```go
// Fantasy Agent.Stream() 的回调接口
OnTextDelta(text)           // 文本增量 → 更新 UI
OnReasoningStart()          // 思考开始 → 显示折叠区域
OnReasoningEnd(content)     // 思考结束 → 填充内容
OnToolCall(call)            // 工具调用 → 显示执行状态
OnToolResult(result)        // 工具结果 → 更新消息
OnStepFinish(reason)        // 步骤完成 → 检查是否继续
```

### 错误处理层级

```
Provider 错误
  ├── 认证失败 → 提示用户检查 API Key
  ├── 余额不足 → 显示充值提示
  ├── 模型不支持 → 提示切换模型
  └── 网络错误 → 自动重试

Agent 错误
  ├── 上下文溢出 → 自动摘要
  ├── 工具执行失败 → 错误消息回传 LLM
  └── 用户取消 → 优雅停止

系统错误
  ├── 数据库错误 → 降级处理
  ├── LSP 崩溃 → 自动重启
  └── MCP 断连 → 自动重连
```

## 会话管理

### 会话生命周期

```
创建 → 活跃 → 暂停/恢复 → 归档
  │                 │
  │                 └─ 上下文自动压缩
  └─ 持久化到 SQLite
```

### 会话队列

同一会话的并发请求会被序列化处理：

```go
// 请求入队
type agentQueue struct {
    queue chan *agentRequest
    // ...
}

// 保证同一会话串行执行
// 新请求排队等待当前完成
// 自动摘要时暂停队列处理
```

### Todo 系统

会话内置任务追踪：
```go
type Todo struct {
    ID     string
    Status string  // pending | in_progress | completed
    Subject string
}
```

通过 `todos` 工具管理，持久化到会话上下文中，摘要时会保留。

## 权限系统

### 三层权限检查

```
Layer 1: Hook 系统预检
  → PreToolUse Hook 可 Allow/Deny/Rewrite

Layer 2: 内置安全检查
  → Bash: 禁用命令列表 (sudo, rm -rf, etc.)
  → Edit: 必须先查看才能编辑
  → Write: 路径安全检查

Layer 3: 用户确认
  → 弹出权限对话框
  → 支持记住选择
```

### 权限请求流

```
Tool.Run()
  → permission.Request(CreatePermissionRequest)
    → Hook 预检 (Allow → 跳过用户确认)
    → 检查已记住的选择
    → 显示权限对话框
      → 用户选择: Allow / Deny / Allow Always
    → 返回结果
```

## LSP 集成

### 懒初始化策略

```
首次访问 .go 文件 → 启动 gopls
首次访问 .ts 文件 → 启动 typescript-language-server
...
```

LSP 客户端按语言维护，首次需要时才启动。

### LSP 提供的能力

| 能力 | 工具 | 用途 |
|------|------|------|
| 诊断 | `diagnostics` | 实时代码错误/警告 |
| 引用 | `references` | 查找符号引用 |
| 悬停 | 内部使用 | 代码信息提示 |
| 文件追踪 | 内部使用 | 检测文件变更 |

LSP 信息自动注入到 Agent 上下文，让 AI 在编码时有代码智能辅助。
