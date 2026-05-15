# 07 - Agent 与 Prompt 工程分析

## Prompt 模板体系

### 模板文件索引

| 模板 | 路径 | Agent 类型 | 用途 |
|------|------|-----------|------|
| `coder.md.tpl` | `internal/agent/templates/` | AgentCoder | 主编程助手系统提示 |
| `task.md.tpl` | `internal/agent/templates/` | AgentTask | 任务分析/项目初始化 |
| `initialize.md.tpl` | `internal/agent/templates/` | 通用 | 基础初始化 |
| `summary.md` | `internal/agent/templates/` | 专用 | 会话摘要生成 |
| `title.md` | `internal/agent/templates/` | 专用 | 标题生成 |
| `agentic_fetch.md` | `internal/agent/templates/` | 工具描述 | AI 增强获取 |
| `agent_tool.md` | `internal/agent/templates/` | 工具描述 | 子 Agent 工具 |

## Coder Agent 系统提示分析

`coder.md.tpl` 是最核心的提示模板，定义了 Crush 作为编程助手的全部行为规范。

### 提示结构

```
1. 身份定义     → 你是谁，你能做什么
2. 通用规则     → 行为准则和约束
3. 通信风格     → 如何与用户沟通
4. 工作流程     → 遇到任务时的标准流程
5. 决策框架     → 如何做出技术选择
6. 文件编辑规范 → 如何修改代码
7. 测试规范     → 如何处理测试
8. 工具使用指南 → 如何使用各种工具
9. 安全约束     → 安全相关的硬性规则
```

### 核心行为规则

1. **自主性要求**: 在用户请求的范围内自主行动，不要不必要地确认
2. **精确匹配**: 编辑文件时必须精确匹配原始文本
3. **安全优先**: 避免引入安全漏洞
4. **最小变更**: 只做必要的修改，不额外重构
5. **测试驱动**: 修改功能后确保测试通过

### 模板变量注入

```
{{.WorkingDir}}        → 当前工作目录
{{.Environment}}       → 环境信息
{{.Shell}}             → 用户 Shell
{{.Platform}}          → 操作系统
{{.MemoryFile}}        → 记忆文件内容
{{.LSPServers}}        → LSP 服务器列表
{{.MCPServers}}        → MCP 服务器列表
```

## Agent 类型

### AgentCoder — 编程助手

```
职责: 主要的编程 Agent
工具: 全部工具集
模型: 用户选择的大模型
提示: coder.md.tpl
特点:
  - 完整的工具访问权限
  - 支持多轮对话
  - 自动上下文管理
  - 支持子 Agent 委派
```

### AgentTask — 任务分析

```
职责: 代码库分析和项目初始化
工具: 有限（分析工具为主）
模型: 大模型
提示: task.md.tpl
特点:
  - 专注于代码理解
  - 生成项目文档
  - 不执行代码修改
```

### Summary Agent — 摘要生成

```
职责: 会话自动摘要
工具: 无
模型: 小模型（成本优化）
提示: summary.md
特点:
  - 压缩对话历史
  - 保留关键信息
  - 上下文窗口不足时触发
```

### Title Agent — 标题生成

```
职责: 为会话生成简短标题
工具: 无
模型: 小模型
提示: title.md
特点:
  - 极简输出（一行标题）
  - 保持用户语言
  - 基于首条消息生成
```

## 模型策略

### Large Model (大模型)

```
用途: 主要推理和代码生成
配置: config.Models["large"]
触发: 正常对话、复杂任务
特点:
  - 更大的上下文窗口
  - 更强的推理能力
  - 更高的成本
```

### Small Model (小模型)

```
用途: 快速响应和辅助任务
配置: config.Models["small"]
触发: 摘要生成、标题生成
特点:
  - 更快的响应速度
  - 更低的成本
  - 适合简单任务
```

### 模型选择逻辑

```
buildAgentModels(isSubAgent):
  1. 获取全局大/小模型配置
  2. 如果是子 Agent，使用子 Agent 配置覆盖
  3. 构建模型实例（Provider + 参数）
  4. 返回 (largeModel, smallModel)

使用场景:
  → 主对话: 使用 largeModel
  → 摘要生成: 使用 smallModel
  → 标题生成: 使用 smallModel
  → 子 Agent: 使用指定模型
```

## 上下文构建

### 系统提示组装

```
1. 渲染模板 (Go template)
   → 注入工作目录、环境信息等变量

2. 附加记忆文件
   → ~/.claude/CLAUDE.md 内容（如果存在）

3. 附加项目上下文
   → .crush.json 中的 instructions
   → AGENTS.md 内容

4. 附加 LSP/MCP 信息
   → 可用的 LSP 服务器列表
   → 可用的 MCP 工具列表
```

### 消息历史管理

```
构建顺序:
  1. SystemMessage (系统提示)
  2. SummaryMessage (如果存在摘要)
  3. History Messages (对话历史)
  4. UserMessage (当前用户输入)
  5. Tool Messages (待处理的工具结果)

去重和过滤:
  - 过滤孤儿工具结果（无对应工具调用）
  - 去重相同 ID 的消息
  - 确保消息顺序正确
```

## 子 Agent 系统

### Agent Tool

主 Agent 可以启动子 Agent 执行复杂搜索：

```
主 Agent → 调用 agent_tool
  → 启动新的 SessionAgent
  → 子 Agent 可用工具: glob, grep, ls, view
  → 子 Agent 执行搜索任务
  → 返回结果给主 Agent
```

### 与主 Agent 的区别

```
子 Agent:
  - 有限的工具集（只读工具）
  - 独立的对话上下文
  - 结果回传给主 Agent
  - 不直接与用户交互
  - 用于复杂的多步搜索
```

## 摘要系统详解

### 触发条件

```go
// 大模型 (>200K 上下文)
remainingTokens <= 20_000

// 小模型
remainingTokens <= contextWindow * 0.2
```

### 摘要流程

```
1. 暂停当前请求处理
2. 提取当前 Todo 列表
3. 构建摘要请求（conversation history + todos）
4. 调用 Summary Agent（使用小模型）
5. 生成结构化摘要:
   - Current state and progress
   - Files and changes made
   - Technical context and decisions
   - Strategy and next steps
   - Exact actionable items
6. 替换历史消息为摘要消息
7. 恢复请求处理
```

### 摘要质量保证

- 摘要包含**具体的文件路径**和**技术决策**
- 保留**可执行项**，不丢失任务状态
- Todo 列表单独注入，确保任务追踪连续
- 摘要后 Agent 能无缝继续工作

## Prompt 工程技巧

### 1. 角色锚定
通过系统提示明确定义 AI 的角色和行为边界，减少歧义。

### 2. 结构化输出
通过模板定义输出的结构和格式，确保一致性。

### 3. 工具描述即 Prompt
每个工具的 `Info().Description` 也是 Prompt 的一部分，LLM 根据描述决定何时使用。

### 4. 上下文压缩
自动摘要机制解决长对话的上下文窗口限制。

### 5. 分层模型
大模型处理复杂任务，小模型处理辅助任务，平衡成本和质量。
