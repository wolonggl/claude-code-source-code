# Claude Code v2.1.88 Code Wiki

## 1. 项目概述

Claude Code v2.1.88 是 Anthropic 开发的 AI 辅助编程工具，提供强大的代码生成、分析和执行能力。该项目是从 npm 包 `@anthropic-ai/claude-code` 版本 2.1.88 中提取的未捆绑 TypeScript 源代码，用于技术研究、学习和教育交流。

### 主要功能
- AI 辅助代码生成与分析
- 交互式 REPL 环境
- 多种工具集成（文件操作、Shell 命令、Web 搜索等）
- 多代理协作系统
- 上下文管理与压缩
- MCP (Model Context Protocol) 集成
- 远程控制与桌面应用集成

### 技术栈
- TypeScript
- Node.js (>= 18.0.0)
- React/Ink (终端 UI)
- esbuild (构建工具)

## 2. 目录结构

Claude Code 项目采用模块化架构，代码组织清晰，主要目录结构如下：

```
src/
├── main.tsx                 # REPL 引导，4,683 行
├── QueryEngine.ts           # SDK/无头查询生命周期引擎
├── query.ts                 # 主代理循环（785KB，最大文件）
├── Tool.ts                  # 工具接口 + buildTool 工厂
├── Task.ts                  # 任务类型，ID，状态基础
├── tools.ts                 # 工具注册表，预设，过滤
├── commands.ts              # Slash 命令定义
├── context.ts               # 用户输入上下文
├── cost-tracker.ts          # API 成本累积
├── setup.ts                 # 首次运行设置流程
│
├── bridge/                  # Claude Desktop / 远程桥接
├── cli/                     # CLI 基础设施
├── commands/                # ~80 个 slash 命令
├── components/              # React/Ink 终端 UI
├── entrypoints/             # 应用入口点
├── hooks/                   # React 钩子
├── services/                # 业务逻辑层
├── state/                   # 应用状态
├── tasks/                   # 任务实现
├── tools/                   # 40+ 工具实现
├── types/                   # 类型定义
├── utils/                   # 工具函数（最大目录）
└── vendor/                  # 原生模块源存根
```

### 核心目录说明

| 目录 | 主要职责 | 文件位置 |
|------|---------|----------|
| entrypoints | 应用入口点，包括 CLI 和 SDK | [src/entrypoints](file:///workspace/src/entrypoints) |
| services | 业务逻辑层，包括 API 客户端、分析、压缩等 | [src/services](file:///workspace/src/services) |
| tools | 40+ 工具实现，如 BashTool、FileReadTool 等 | [src/tools](file:///workspace/src/tools) |
| utils | 工具函数，包含大量辅助功能 | [src/utils](file:///workspace/src/utils) |
| components | React/Ink 终端 UI 组件 | [src/components](file:///workspace/src/components) |
| bridge | Claude Desktop 和远程连接桥接 | [src/bridge](file:///workspace/src/bridge) |

## 3. 系统架构

Claude Code 采用分层架构设计，从入口层到工具系统、服务层和状态层，形成完整的代理循环。

### 架构层次

```
┌─────────────────────────────────────────────────────────────────────┐
│                         ENTRY LAYER                                 │
│  cli.tsx ──> main.tsx ──> REPL.tsx (interactive)                   │
│                     └──> QueryEngine.ts (headless/SDK)              │
└──────────────────────────────┬──────────────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────────────┐
│                       QUERY ENGINE                                  │
│  submitMessage(prompt) ──> AsyncGenerator<SDKMessage>               │
│    │                                                                │
│    ├── fetchSystemPromptParts()    ──> assemble system prompt       │
│    ├── processUserInput()          ──> handle /commands             │
│    ├── query()                     ──> main agent loop              │
│    │     ├── StreamingToolExecutor ──> parallel tool execution       │
│    │     ├── autoCompact()         ──> context compression          │
│    │     └── runTools()            ──> tool orchestration           │
│    └── yield SDKMessage            ──> stream to consumer           │
└──────────────────────────────┬──────────────────────────────────────┘
                               │
              ┌────────────────┼────────────────┐
              ▼                ▼                 ▼
┌──────────────────┐ ┌─────────────────┐ ┌──────────────────┐
│   TOOL SYSTEM    │ │  SERVICE LAYER  │ │   STATE LAYER    │
│                  │ │                 │ │                  │
│ Tool Interface   │ │ api/claude.ts   │ │ AppState Store   │
│  ├─ call()       │ │  API client     │ │  ├─ permissions  │
│  ├─ validate()   │ │ compact/        │ │  ├─ fileHistory  │
│  ├─ checkPerms() │ │  auto-compact   │ │  ├─ agents       │
│  ├─ render()     │ │ mcp/            │ │  └─ fastMode     │
│  └─ prompt()     │ │  MCP protocol   │ │                  │
│                  │ │ analytics/      │ │ React Context    │
│ 40+ Built-in:    │ │  telemetry      │ │  ├─ useAppState  │
│  ├─ BashTool     │ │ tools/          │ │  └─ useSetState  │
│  ├─ FileRead     │ │  executor       │ │                  │
│  ├─ FileEdit     │ │ plugins/        │ └──────────────────┘
│  ├─ Glob/Grep    │ │  loader         │
│  ├─ AgentTool    │ │ settingsSync/   │
│  ├─ WebFetch     │ │  cross-device   │
│  └─ MCPTool      │ │ oauth/          │
│                  │ │  auth flow      │
└──────────────────┘ └─────────────────┘
              │                │
              ▼                ▼
┌──────────────────┐ ┌─────────────────┐
│   TASK SYSTEM    │ │   BRIDGE LAYER  │
│                  │ │                 │
│ Task Types:      │ │ bridgeMain.ts   │
│  ├─ local_bash   │ │  session mgmt   │
│  ├─ local_agent  │ │ bridgeApi.ts    │
│  ├─ remote_agent │ │  HTTP client    │
│  ├─ in_process   │ │ workSecret.ts   │
│  ├─ dream        │ │  auth tokens    │
│  └─ workflow     │ │ sessionRunner   │
│                  │ │  process spawn  │
│ ID: prefix+8chr  │ └─────────────────┘
│  b=bash a=agent  │
│  r=remote t=team │
└──────────────────┘
```

### 数据流：单个查询生命周期

```
 USER INPUT (prompt / slash command)
     │
     ▼
 processUserInput()                ← parse /commands, build UserMessage
     │
     ▼
 fetchSystemPromptParts()          ← tools → prompt sections, CLAUDE.md memory
     │
     ▼
 recordTranscript()                ← persist user message to disk (JSONL)
     │
     ▼
 ┌─→ normalizeMessagesForAPI()     ← strip UI-only fields, compact if needed
 │   │
 │   ▼
 │   Claude API (streaming)        ← POST /v1/messages with tools + system prompt
 │   │
 │   ▼
 │   stream events                 ← message_start → content_block_delta → message_stop
 │   │
 │   ├─ text block ──────────────→ yield to consumer (SDK / REPL)
 │   │
 │   └─ tool_use block?
 │       │
 │       ▼
 │   StreamingToolExecutor         ← partition: concurrent-safe vs serial
 │       │
 │       ▼
 │   canUseTool()                  ← permission check (hooks + rules + UI prompt)
 │       │
 │       ├─ DENY ────────────────→ append tool_result(error), continue loop
 │       │
 │       └─ ALLOW
 │           │
 │           ▼
 │       tool.call()               ← execute the tool (Bash, Read, Edit, etc.)
 │           │
 │           ▼
 │       append tool_result        ← push to messages[], recordTranscript()
 │           │
 └─────────┘                      ← loop back to API call
     │
     ▼ (stop_reason != "tool_use")
 yield result message              ← final text, usage, cost, session_id
```

## 4. 核心模块

### 4.1 QueryEngine

QueryEngine 是 Claude Code 的核心查询引擎，负责管理查询生命周期和会话状态。它从 ask() 中提取核心逻辑到一个独立的类中，可以被无头/SDK 路径和 REPL 使用。

**主要功能：**
- 管理会话状态和消息历史
- 处理用户输入和命令
- 构建系统提示
- 执行查询循环
- 处理工具使用和权限
- 流式输出结果

**关键方法：**
- `submitMessage(prompt)`: 提交用户提示并返回异步生成器
- `interrupt()`: 中断当前查询
- `getMessages()`: 获取当前消息历史

**文件位置：** [src/QueryEngine.ts](file:///workspace/src/QueryEngine.ts)

### 4.2 主查询循环 (query.ts)

主查询循环是 Claude Code 的核心代理循环，处理与 Claude API 的交互、工具执行和上下文管理。这是项目中最大的文件（785KB），包含了复杂的代理逻辑。

**主要功能：**
- 处理消息和系统提示
- 调用 Claude API 进行推理
- 执行工具调用（支持流式执行）
- 管理上下文压缩（自动压缩、微压缩、上下文折叠）
- 处理错误和重试（包括模型回退）
- 管理任务预算和令牌使用
- 生成工具使用摘要
- 处理内存预取和技能发现
- 管理命令队列和附件

**关键流程：**
1. 准备消息和系统提示
2. 应用上下文处理（微压缩、上下文折叠、自动压缩）
3. 调用 Claude API 获取响应
4. 处理工具使用请求
5. 执行工具并获取结果
6. 生成工具使用摘要
7. 处理内存预取和技能发现结果
8. 检查任务预算和令牌使用
9. 循环直到完成

**文件位置：** [src/query.ts](file:///workspace/src/query.ts)

### 4.3 工具系统 (Tool.ts)

工具系统是 Claude Code 的扩展机制，允许模型执行各种操作，如文件操作、Shell 命令、Web 搜索等。

**主要功能：**
- 定义工具接口和生命周期
- 提供工具构建工厂
- 管理工具权限和验证
- 支持工具并发执行

**关键组件：**
- `Tool` 接口：定义工具的行为和能力
- `buildTool()`: 构建工具实例的工厂函数
- `Tools` 类型：工具集合

**文件位置：** [src/Tool.ts](file:///workspace/src/Tool.ts)

### 4.4 命令系统

命令系统提供了一系列 slash 命令，用于控制 Claude Code 的行为和功能。

**主要功能：**
- 处理用户输入的命令
- 执行各种操作，如配置、计划模式、会话管理等
- 提供命令行界面

**文件位置：** [src/commands.ts](file:///workspace/src/commands.ts) 和 [src/commands](file:///workspace/src/commands)

### 4.5 状态管理

状态管理系统维护应用的全局状态，包括权限、文件历史、代理状态等。

**主要功能：**
- 管理应用状态
- 提供状态更新和订阅机制
- 处理权限和设置

**文件位置：** [src/state](file:///workspace/src/state)

## 5. 关键类与函数

### 5.1 QueryEngine 类

**功能：** 管理查询生命周期和会话状态

**关键方法：**
- `submitMessage(prompt, options)`: 提交用户提示并返回异步生成器
- `interrupt()`: 中断当前查询
- `getMessages()`: 获取当前消息历史
- `getReadFileState()`: 获取文件读取状态缓存
- `getSessionId()`: 获取会话 ID
- `setModel(model)`: 设置模型

**文件位置：** [src/QueryEngine.ts](file:///workspace/src/QueryEngine.ts)

### 5.2 buildTool 函数

**功能：** 从工具定义构建完整的工具实例，填充默认值

**参数：**
- `def`: 工具定义对象

**返回值：** 完整的 Tool 实例

**默认值：**
- `isEnabled`: true
- `isConcurrencySafe`: false
- `isReadOnly`: false
- `isDestructive`: false
- `checkPermissions`: 允许操作
- `toAutoClassifierInput`: 空字符串
- `userFacingName`: 工具名称

**文件位置：** [src/Tool.ts](file:///workspace/src/Tool.ts)

### 5.3 query 函数

**功能：** 主代理循环，处理与 Claude API 的交互、工具执行和上下文管理

**参数：**
- `params`: 查询参数，包含以下字段：
  - `messages`: 消息数组
  - `systemPrompt`: 系统提示
  - `userContext`: 用户上下文对象
  - `systemContext`: 系统上下文对象
  - `canUseTool`: 工具使用权限检查函数
  - `toolUseContext`: 工具使用上下文
  - `fallbackModel`: 回退模型（可选）
  - `querySource`: 查询来源
  - `maxOutputTokensOverride`: 最大输出令牌覆盖（可选）
  - `maxTurns`: 最大轮数（可选）
  - `skipCacheWrite`: 是否跳过缓存写入（可选）
  - `taskBudget`: 任务预算（可选）
  - `deps`: 查询依赖（可选）

**返回值：** 异步生成器，产生以下类型的事件：
- `StreamEvent`: 流事件
- `RequestStartEvent`: 请求开始事件
- `Message`: 消息
- `TombstoneMessage`: 墓碑消息（用于移除UI中的消息）
- `ToolUseSummaryMessage`: 工具使用摘要消息

**核心逻辑：**
1. **初始化状态**：设置初始状态，包括消息、工具使用上下文等
2. **上下文处理**：应用微压缩、上下文折叠和自动压缩
3. **API调用**：调用Claude API获取响应
4. **工具执行**：处理工具使用请求并执行工具
5. **错误处理**：处理各种错误情况，包括模型回退
6. **摘要生成**：生成工具使用摘要
7. **内存预取**：处理内存预取结果
8. **技能发现**：处理技能发现结果
9. **任务预算**：检查任务预算和令牌使用
10. **循环继续**：根据需要继续下一轮循环

**文件位置：** [src/query.ts](file:///workspace/src/query.ts)

### 5.4 BashTool

**功能：** 执行 Shell 命令的工具

**关键方法：**
- `call(input, context, canUseTool, parentMessage, onProgress)`: 执行 Shell 命令
- `isSearchOrReadCommand(input)`: 检查命令是否为搜索或读取操作
- `validateInput(input)`: 验证输入
- `checkPermissions(input, context)`: 检查权限

**文件位置：** [src/tools/BashTool/BashTool.tsx](file:///workspace/src/tools/BashTool/BashTool.tsx)

### 5.5 StreamingToolExecutor

**功能：** 并行执行工具调用

**关键方法：**
- `addTool(toolBlock, message)`: 添加工具调用
- `getCompletedResults()`: 获取已完成的结果
- `getRemainingResults()`: 获取剩余的结果
- `discard()`: 丢弃所有结果

**文件位置：** [src/services/tools/StreamingToolExecutor.js](file:///workspace/src/services/tools/StreamingToolExecutor.js)

### 5.6 runTools 函数

**功能：** 工具编排，执行多个工具调用

**参数：**
- `toolUses`: 工具使用数组
- `tools`: 可用工具
- `canUseTool`: 权限检查函数
- `context`: 工具使用上下文

**返回值：** 工具结果数组

**文件位置：** [src/services/tools/toolOrchestration.js](file:///workspace/src/services/tools/toolOrchestration.js)

### 5.7 autoCompact 函数

**功能：** 自动压缩上下文，当 token 计数超过阈值时触发

**参数：**
- `messages`: 消息数组
- `context`: 工具使用上下文
- `params`: 压缩参数
- `querySource`: 查询来源
- `tracking`: 跟踪状态
- `snipTokensFreed`: 剪切释放的 tokens

**返回值：** 压缩结果

**文件位置：** [src/services/compact/autoCompact.js](file:///workspace/src/services/compact/autoCompact.js)

### 5.8 processUserInput 函数

**功能：** 处理用户输入，解析命令和构建用户消息

**参数：**
- `input`: 用户输入
- `mode`: 模式
- `setToolJSX`: 设置工具 JSX
- `context`: 上下文
- `messages`: 消息数组
- `uuid`: UUID
- `isMeta`: 是否为元消息
- `querySource`: 查询来源

**返回值：** 处理结果，包括消息、是否查询、允许的工具等

**文件位置：** [src/utils/processUserInput/processUserInput.js](file:///workspace/src/utils/processUserInput/processUserInput.js)

### 5.9 fetchSystemPromptParts 函数

**功能：** 获取系统提示部分，包括工具、权限和内存

**参数：**
- `tools`: 工具数组
- `mainLoopModel`: 主循环模型
- `additionalWorkingDirectories`: 额外工作目录
- `mcpClients`: MCP 客户端
- `customSystemPrompt`: 自定义系统提示

**返回值：** 系统提示部分

**文件位置：** [src/utils/queryContext.js](file:///workspace/src/utils/queryContext.js)

### 5.10 recordTranscript 函数

**功能：** 记录会话转录到磁盘

**参数：**
- `messages`: 消息数组
- `agentId`: 代理 ID

**返回值：** Promise

**文件位置：** [src/utils/sessionStorage.js](file:///workspace/src/utils/sessionStorage.js)

## 6. 工具系统

Claude Code 包含 40+ 内置工具，分为多个类别：

### 6.1 文件操作工具

| 工具 | 功能 | 文件位置 |
|------|------|----------|
| FileReadTool | 读取文件内容 | [src/tools/FileReadTool](file:///workspace/src/tools/FileReadTool) |
| FileEditTool | 编辑文件内容 | [src/tools/FileEditTool](file:///workspace/src/tools/FileEditTool) |
| FileWriteTool | 创建新文件 | [src/tools/FileWriteTool](file:///workspace/src/tools/FileWriteTool) |
| NotebookEditTool | 编辑笔记本 | [src/tools/NotebookEditTool](file:///workspace/src/tools/NotebookEditTool) |

### 6.2 搜索与发现工具

| 工具 | 功能 | 文件位置 |
|------|------|----------|
| GlobTool | 文件模式搜索 | [src/tools/GlobTool](file:///workspace/src/tools/GlobTool) |
| GrepTool | 内容搜索 | [src/tools/GrepTool](file:///workspace/src/tools/GrepTool) |
| ToolSearchTool | 工具搜索 | [src/tools/ToolSearchTool](file:///workspace/src/tools/ToolSearchTool) |

### 6.3 执行工具

| 工具 | 功能 | 文件位置 |
|------|------|----------|
| BashTool | 执行 Shell 命令 | [src/tools/BashTool](file:///workspace/src/tools/BashTool) |
| PowerShellTool | 执行 PowerShell 命令 | [src/tools/PowerShellTool](file:///workspace/src/tools/PowerShellTool) |

### 6.4 Web 与网络工具

| 工具 | 功能 | 文件位置 |
|------|------|----------|
| WebFetchTool | HTTP 获取 | [src/tools/WebFetchTool](file:///workspace/src/tools/WebFetchTool) |
| WebSearchTool | Web 搜索 | [src/tools/WebSearchTool](file:///workspace/src/tools/WebSearchTool) |

### 6.5 代理与任务工具

| 工具 | 功能 | 文件位置 |
|------|------|----------|
| AgentTool | 子代理生成 | [src/tools/AgentTool](file:///workspace/src/tools/AgentTool) |
| SendMessageTool | 代理间消息 | [src/tools/SendMessageTool](file:///workspace/src/tools/SendMessageTool) |
| TaskCreateTool | 创建任务 | [src/tools/TaskCreateTool](file:///workspace/src/tools/TaskCreateTool) |
| TaskGetTool | 获取任务 | [src/tools/TaskGetTool](file:///workspace/src/tools/TaskGetTool) |
| TaskListTool | 列出任务 | [src/tools/TaskListTool](file:///workspace/src/tools/TaskListTool) |
| TaskStopTool | 停止任务 | [src/tools/TaskStopTool](file:///workspace/src/tools/TaskStopTool) |
| TaskUpdateTool | 更新任务 | [src/tools/TaskUpdateTool](file:///workspace/src/tools/TaskUpdateTool) |
| TaskOutputTool | 任务输出 | [src/tools/TaskOutputTool](file:///workspace/src/tools/TaskOutputTool) |

### 6.6 MCP 协议工具

| 工具 | 功能 | 文件位置 |
|------|------|----------|
| MCPTool | MCP 工具包装器 | [src/tools/MCPTool](file:///workspace/src/tools/MCPTool) |
| ListMcpResourcesTool | 列出 MCP 资源 | [src/tools/ListMcpResourcesTool](file:///workspace/src/tools/ListMcpResourcesTool) |
| ReadMcpResourceTool | 读取 MCP 资源 | [src/tools/ReadMcpResourceTool](file:///workspace/src/tools/ReadMcpResourceTool) |

### 6.7 技能与扩展工具

| 工具 | 功能 | 文件位置 |
|------|------|----------|
| SkillTool | 技能调用 | [src/tools/SkillTool](file:///workspace/src/tools/SkillTool) |
| LSPTool | 语言服务器协议 | [src/tools/LSPTool](file:///workspace/src/tools/LSPTool) |

### 6.8 规划与工作流工具

| 工具 | 功能 | 文件位置 |
|------|------|----------|
| EnterPlanModeTool | 进入计划模式 | [src/tools/EnterPlanModeTool](file:///workspace/src/tools/EnterPlanModeTool) |
| ExitPlanModeTool | 退出计划模式 | [src/tools/ExitPlanModeTool](file:///workspace/src/tools/ExitPlanModeTool) |
| EnterWorktreeTool | 进入工作树 | [src/tools/EnterWorktreeTool](file:///workspace/src/tools/EnterWorktreeTool) |
| ExitWorktreeTool | 退出工作树 | [src/tools/ExitWorktreeTool](file:///workspace/src/tools/ExitWorktreeTool) |
| TodoWriteTool | 写入待办事项 | [src/tools/TodoWriteTool](file:///workspace/src/tools/TodoWriteTool) |

### 6.9 系统工具

| 工具 | 功能 | 文件位置 |
|------|------|----------|
| ConfigTool | 配置管理 | [src/tools/ConfigTool](file:///workspace/src/tools/ConfigTool) |
| ScheduleCronTool | 调度 cron 任务 | [src/tools/ScheduleCronTool](file:///workspace/src/tools/ScheduleCronTool) |
| SleepTool | 睡眠/延迟 | [src/tools/SleepTool](file:///workspace/src/tools/SleepTool) |

## 7. 依赖关系

Claude Code 的依赖非常简洁，主要包含开发依赖：

| 依赖 | 版本 | 用途 |
|------|------|------|
| esbuild | ^0.27.4 | 构建工具 |
| typescript | ^6.0.2 | TypeScript 支持 |

**运行时要求：**
- Node.js >= 18.0.0

**注意：** 该项目原本使用 Bun 运行时的编译时内在函数（如 `feature()` 和 `MACRO`），但在构建过程中会被替换为运行时值。

## 8. 运行与构建

### 8.1 构建流程

Claude Code 的构建流程包括以下步骤：

1. **准备源代码** (`npm run prepare-src`):
   - 替换 `import { feature } from 'bun:bundle'` 为存根
   - 替换 `MACRO.X` 引用为运行时值
   - 创建缺失的类型声明

2. **构建** (`npm run build`):
   - 复制 src/ 到 build-src/
   - 替换 `feature('X')` 为 `false`
   - 替换 `MACRO.VERSION` 等为字符串字面量
   - 创建缺失的特性门控模块的存根
   - 使用 esbuild 打包为 dist/cli.js

3. **类型检查** (`npm run check`):
   - 运行 TypeScript 类型检查

### 8.2 运行方式

```bash
# 构建项目
npm run build

# 运行构建后的 CLI
node dist/cli.js

# 查看版本
node dist/cli.js --version

# 执行简单提示
node dist/cli.js -p "Hello"
```

### 8.3 开发流程

1. **安装依赖**:
   ```bash
   npm install
   ```

2. **准备源代码**:
   ```bash
   npm run prepare-src
   ```

3. **类型检查**:
   ```bash
   npm run check
   ```

4. **构建**:
   ```bash
   npm run build
   ```

## 9. 配置与环境

### 9.1 环境变量

| 环境变量 | 描述 | 默认值 |
|----------|------|--------|
| NODE_ENV | 运行环境 | 未设置 |
| USER_TYPE | 用户类型 | 未设置（'ant' 为 Anthropic 内部） |
| CLAUDE_CODE_DISABLE_BACKGROUND_TASKS | 禁用后台任务 | 未设置 |
| CLAUDE_CODE_EAGER_FLUSH | 启用急切刷新 | 未设置 |
| CLAUDE_CODE_IS_COWORK | 是否为 Cowork 模式 | 未设置 |
| MAX_STRUCTURED_OUTPUT_RETRIES | 结构化输出最大重试次数 | 5 |

### 9.2 配置文件

Claude Code 使用 settings.json 文件存储配置，包括：
- 工具权限规则
- 主题设置
- 模型设置
- MCP 服务器配置
- 插件配置

### 9.3 特性标志

Claude Code 使用 `feature()` 函数进行特性门控，主要特性包括：

| 特性标志 | 描述 |
|----------|------|
| COORDINATOR_MODE | 多代理协调器 |
| HISTORY_SNIP | 积极的历史修剪 |
| CONTEXT_COLLAPSE | 上下文重组 |
| DAEMON | 后台守护进程工作器 |
| KAIROS | 推送通知，文件发送 |
| PROACTIVE | 睡眠工具，主动行为 |
| WEB_BROWSER_TOOL | 浏览器自动化 |
| VOICE_MODE | 语音输入/输出 |
| EXPERIMENTAL_SKILL_SEARCH | 技能发现 |

## 10. 总结

Claude Code v2.1.88 是一个功能强大的 AI 辅助编程工具，具有以下特点：

### 核心优势

1. **模块化架构**：清晰的分层设计，易于理解和扩展
2. **强大的工具系统**：40+ 内置工具，支持文件操作、Shell 命令、Web 搜索等
3. **多代理协作**：支持子代理、远程代理和团队协作
4. **上下文管理**：智能压缩和管理上下文，优化 token 使用
5. **MCP 集成**：支持 Model Context Protocol，扩展能力
6. **远程控制**：与 Claude Desktop 和远程系统集成

### 技术亮点

1. **AsyncGenerator 流式处理**：从 API 到消费者的全链流式处理
2. **Builder + Factory 模式**：工具定义的安全默认值
3. **特性标志 + DCE**：Bun 编译时死代码消除
4. **鉴别联合类型**：类型安全的消息处理
5. **观察者 + 状态机**：工具执行生命周期跟踪
6. **快照状态**：文件操作的撤销/重做

### 应用场景

1. **代码生成与分析**：智能生成和分析代码
2. **交互式编程**：通过 REPL 环境进行交互式开发
3. **自动化任务**：执行 Shell 命令和脚本
4. **多代理协作**：分解复杂任务，多代理并行处理
5. **远程开发**：通过远程连接进行开发

Claude Code v2.1.88 展示了如何构建一个生产级的 AI 代理系统，通过多层机制和工具集成，提供了强大而灵活的编程辅助能力。