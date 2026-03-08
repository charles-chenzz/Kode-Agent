# AI Agent 核心流程详解

> 本文档深入解析 Kode 的核心 - Agentic Loop（代理循环）的实现原理。

## 一、什么是 Agentic Loop？

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        Agentic Loop 本质                                    │
│                                                                             │
│   传统程序：                                                                 │
│   ┌─────────────────────────────────────────────────────────────────────┐  │
│   │  Input → [固定逻辑] → Output                                        │  │
│   │                                                                     │  │
│   │  程序员预先写好所有逻辑，输入确定后输出就确定                        │  │
│   └─────────────────────────────────────────────────────────────────────┘  │
│                                                                             │
│   AI Agent：                                                                │
│   ┌─────────────────────────────────────────────────────────────────────┐  │
│   │  Input → [AI 决策] → Action → Observation → [AI 决策] → ...         │  │
│   │                                                                     │  │
│   │  AI 根据当前状态决定下一步做什么，形成循环                          │  │
│   └─────────────────────────────────────────────────────────────────────┘  │
│                                                                             │
│   核心区别：决策权从程序员转移到了 AI 模型                                   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

## 二、核心代码位置

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           核心文件                                          │
│                                                                             │
│   src/app/query.ts                                                          │
│   ┌─────────────────────────────────────────────────────────────────────┐  │
│   │                                                                     │  │
│   │   query()      - 主入口，对外暴露的查询函数                         │  │
│   │   queryCore()  - 核心循环实现                                       │  │
│   │   runToolUse() - 单个工具的执行                                     │  │
│   │                                                                     │  │
│   └─────────────────────────────────────────────────────────────────────┘  │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

## 三、完整流程图

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        Agentic Loop 完整流程                                │
│                                                                             │
│   用户输入: "帮我重构 auth.ts 文件，添加错误处理"                           │
│       │                                                                     │
│       ▼                                                                     │
│   ┌─────────────────────────────────────────────────────────────────────┐  │
│   │  1. query() 入口                                                    │  │
│   │     • 接收消息列表、系统提示、上下文                                │  │
│   │     • 持久化用户消息到会话日志                                      │  │
│   └─────────────────────────────────────────────────────────────────────┘  │
│       │                                                                     │
│       ▼                                                                     │
│   ┌─────────────────────────────────────────────────────────────────────┐  │
│   │  2. queryCore() 核心处理                                            │  │
│   │     • 设置请求状态为 'thinking'                                     │  │
│   │     • 检查是否需要自动压缩会话                                      │  │
│   │     • 处理后台 Shell 通知                                           │  │
│   │     • 运行 UserPromptSubmit Hooks                                   │  │
│   └─────────────────────────────────────────────────────────────────────┘  │
│       │                                                                     │
│       ▼                                                                     │
│   ┌─────────────────────────────────────────────────────────────────────┐  │
│   │  3. 构建系统提示                                                    │  │
│   │     • 基础系统提示                                                  │  │
│   │     + Kode 上下文 (README, Git状态, 目录结构)                       │  │
│   │     + Plan 模式增强 (如果在计划模式)                                │  │
│   │     + Hook 增强内容                                                 │  │
│   │     + 输出风格增强                                                  │  │
│   │     + 系统提醒                                                      │  │
│   └─────────────────────────────────────────────────────────────────────┘  │
│       │                                                                     │
│       ▼                                                                     │
│   ┌─────────────────────────────────────────────────────────────────────┐  │
│   │  4. 调用 LLM (queryLLM)                                             │  │
│   │     • 消息标准化                                                    │  │
│   │     • 工具 Schema 转换                                              │  │
│   │     • 发送到 AI 模型                                                │  │
│   │     • 流式接收响应                                                  │  │
│   └─────────────────────────────────────────────────────────────────────┘  │
│       │                                                                     │
│       ▼                                                                     │
│   ┌─────────────────────────────────────────────────────────────────────┐  │
│   │  5. 解析 LLM 响应                                                   │  │
│   │                                                                     │  │
│   │     响应可能包含：                                                  │  │
│   │     ┌─────────────────────────────────────────────────────────┐    │  │
│   │     │ [                                                        │    │  │
│   │     │   { type: "text", text: "我来帮你重构..." },            │    │  │
│   │     │   { type: "tool_use", name: "Read", input: {...} },     │    │  │
│   │     │   { type: "tool_use", name: "Edit", input: {...} }      │    │  │
│   │     │ ]                                                        │    │  │
│   │     └─────────────────────────────────────────────────────────┘    │  │
│   └─────────────────────────────────────────────────────────────────────┘  │
│       │                                                                     │
│       ├─────────────────────────────────────────────────────┐              │
│       │                                                     │              │
│       ▼                                                     ▼              │
│   ┌─────────────────┐                              ┌─────────────────┐     │
│   │ 无工具调用      │                              │ 有工具调用      │     │
│   │                 │                              │                 │     │
│   │ 运行 Stop Hooks │                              │ 进入工具队列    │     │
│   │ 输出文本给用户  │                              │                 │     │
│   │ 循环结束        │                              │                 │     │
│   └─────────────────┘                              └────────┬────────┘     │
│                                                              │              │
│                                                              ▼              │
│   ┌─────────────────────────────────────────────────────────────────────┐  │
│   │  6. ToolUseQueue - 工具队列管理                                     │  │
│   │                                                                     │  │
│   │     • 管理多个工具的并发执行                                        │  │
│   │     • 处理工具间的依赖关系                                          │  │
│   │     • 支持进度消息的实时推送                                        │  │
│   │                                                                     │  │
│   │     ┌─────────────────────────────────────────────────────────┐    │  │
│   │     │                                                         │    │  │
│   │     │  Tool 1 (Read)    ──────→  执行中  ──────→  完成       │    │  │
│   │     │  Tool 2 (Read)    ──────→  执行中  ──────→  完成       │    │  │
│   │     │  Tool 3 (Edit)    ──────→  等待    ──────→  执行  ──→  │    │  │
│   │     │                                                         │    │  │
│   │     │  并发安全的工具可以并行，不安全的需要排队               │    │  │
│   │     │                                                         │    │  │
│   │     └─────────────────────────────────────────────────────────┘    │  │
│   └─────────────────────────────────────────────────────────────────────┘  │
│       │                                                                     │
│       ▼                                                                     │
│   ┌─────────────────────────────────────────────────────────────────────┐  │
│   │  7. runToolUse() - 单个工具执行                                     │  │
│   │                                                                     │  │
│   │     a. 解析工具名称和输入                                           │  │
│   │     b. 查找工具定义                                                 │  │
│   │     c. 运行 PreToolUse Hooks                                        │  │
│   │     d. 验证输入 Schema                                              │  │
│   │     e. 检查权限 (canUseTool)                                        │  │
│   │     f. 执行工具 (tool.call)                                         │  │
│   │     g. 运行 PostToolUse Hooks                                       │  │
│   │     h. 返回结果                                                     │  │
│   └─────────────────────────────────────────────────────────────────────┘  │
│       │                                                                     │
│       ▼                                                                     │
│   ┌─────────────────────────────────────────────────────────────────────┐  │
│   │  8. 收集工具结果                                                    │  │
│   │                                                                     │  │
│   │     工具执行结果格式：                                              │  │
│   │     ┌─────────────────────────────────────────────────────────┐    │  │
│   │     │ {                                                        │    │  │
│   │     │   type: "tool_result",                                   │    │  │
│   │     │   tool_use_id: "xxx",                                    │    │  │
│   │     │   content: "文件内容...",  // 或错误信息                 │    │  │
│   │     │   is_error: false         // 是否出错                    │    │  │
│   │     │ }                                                        │    │  │
│   │     └─────────────────────────────────────────────────────────┘    │  │
│   └─────────────────────────────────────────────────────────────────────┘  │
│       │                                                                     │
│       ▼                                                                     │
│   ┌─────────────────────────────────────────────────────────────────────┐  │
│   │  9. 递归调用 queryCore()                                            │  │
│   │                                                                     │  │
│   │     新的消息列表 = 原消息 + AI响应 + 工具结果                       │  │
│   │                                                                     │  │
│   │     AI 现在知道工具执行的结果，可以继续决策                         │  │
│   │                                                                     │  │
│   └─────────────────────────────────────────────────────────────────────┘  │
│       │                                                                     │
│       └──────────────────────────────────────┐                              │
│                                              │                              │
│                    ┌─────────────────────────┘                              │
│                    │                                                        │
│                    ▼                                                        │
│   ┌─────────────────────────────────────────────────────────────────────┐  │
│   │  循环继续...直到 AI 不再调用工具，直接给出最终答案                   │  │
│   └─────────────────────────────────────────────────────────────────────┘  │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

## 四、核心代码解析

### 4.1 query() 函数 - 入口

```typescript
// src/app/query.ts (简化版)

export async function* query(
  messages: Message[],           // 历史消息
  systemPrompt: string[],        // 系统提示
  context: { [k: string]: string },  // 上下文变量
  canUseTool: CanUseToolFn,      // 权限检查函数
  toolUseContext: ExtendedToolUseContext,  // 工具执行上下文
): AsyncGenerator<Message, void> {
  
  // 1. 持久化用户消息
  if (messages.length > 0) {
    const lastMessage = messages[messages.length - 1]
    if (lastMessage.type === 'user') {
      appendSessionJsonlFromMessage({ message: lastMessage, toolUseContext })
    }
  }

  // 2. 委托给核心处理
  for await (const message of queryCore(
    messages,
    systemPrompt,
    context,
    canUseTool,
    toolUseContext,
  )) {
    // 3. 持久化每条输出消息
    appendSessionJsonlFromMessage({ message, toolUseContext })
    yield message
  }
}
```

### 4.2 queryCore() 函数 - 核心循环

```typescript
// src/app/query.ts (简化版)

async function* queryCore(
  messages: Message[],
  systemPrompt: string[],
  context: { [k: string]: string },
  canUseTool: CanUseToolFn,
  toolUseContext: ExtendedToolUseContext,
): AsyncGenerator<Message, void> {
  
  // 1. 设置状态
  setRequestStatus({ kind: 'thinking' })

  // 2. 检查自动压缩
  const { messages: processedMessages, wasCompacted } = 
    await checkAutoCompact(messages, toolUseContext)
  if (wasCompacted) {
    messages = processedMessages
  }

  // 3. 处理后台 Shell 通知
  const shell = BunShell.getInstance()
  const notifications = shell.flushBashNotifications()
  for (const notification of notifications) {
    yield createAssistantMessage(renderBashNotification(notification))
  }

  // 4. 运行用户提示提交 Hooks
  const promptOutcome = await runUserPromptSubmitHooks({
    prompt: userPromptText,
    permissionMode: toolUseContext.options?.toolPermissionContext?.mode,
    cwd: getCwd(),
    // ...
  })
  
  // 如果 Hook 阻止，直接返回
  if (promptOutcome.decision === 'block') {
    yield createAssistantMessage(promptOutcome.message)
    return
  }

  // 5. 构建完整系统提示
  const { systemPrompt: fullSystemPrompt, reminders } = 
    formatSystemPromptWithContext(systemPrompt, context, toolUseContext.agentId)
  
  // 添加各种增强
  fullSystemPrompt.push(...getPlanModeSystemPromptAdditions(messages, toolUseContext))
  fullSystemPrompt.push(...drainHookSystemPromptAdditions(toolUseContext))
  fullSystemPrompt.push(...getOutputStyleSystemPromptAdditions())

  // 6. 调用 LLM
  const assistantMessage = await queryLLM(
    normalizeMessagesForAPI(messages),
    fullSystemPrompt,
    toolUseContext.options.maxThinkingTokens,
    toolUseContext.options.tools,
    toolUseContext.abortController.signal,
    { /* options */ }
  )

  // 7. 输出 AI 响应
  yield assistantMessage

  // 8. 检查是否有工具调用
  const toolUseMessages = assistantMessage.message.content.filter(
    block => block.type === 'tool_use' || 
             block.type === 'server_tool_use' || 
             block.type === 'mcp_tool_use'
  )

  // 9. 如果没有工具调用，运行 Stop Hooks 并结束
  if (!toolUseMessages.length) {
    const stopOutcome = await runStopHooks({ /* ... */ })
    if (stopOutcome.decision === 'block') {
      // 递归调用继续
      yield* await queryCore([...messages, assistantMessage], ...)
    }
    return
  }

  // 10. 创建工具队列并执行
  const toolQueue = new ToolUseQueue({
    toolDefinitions: toolUseContext.options.tools,
    canUseTool,
    toolUseContext,
    siblingToolUseIDs: new Set(toolUseMessages.map(t => t.id)),
  })

  for (const toolUse of toolUseMessages) {
    toolQueue.addTool(toolUse, assistantMessage)
  }

  // 11. 收集工具结果
  const toolMessagesForNextTurn: (UserMessage | AssistantMessage)[] = []
  for await (const message of toolQueue.getRemainingResults()) {
    yield message
    if (message.type !== 'progress') {
      toolMessagesForNextTurn.push(message)
    }
  }

  // 12. 递归调用继续循环
  yield* await queryCore(
    [...messages, assistantMessage, ...toolMessagesForNextTurn],
    systemPrompt,
    context,
    canUseTool,
    toolQueue.getUpdatedContext(),
  )
}
```

### 4.3 ToolUseQueue 类 - 工具队列

```typescript
// src/app/query.ts (简化版)

class ToolUseQueue {
  private tools: ToolQueueEntry[] = []
  
  // 添加工具到队列
  addTool(toolUse: ToolUseBlock, assistantMessage: AssistantMessage) {
    const toolDefinition = this.toolDefinitions.find(t => t.name === toolUse.name)
    const isConcurrencySafe = toolDefinition?.isConcurrencySafe(parsedInput) ?? false
    
    this.tools.push({
      id: toolUse.id,
      block: toolUse,
      assistantMessage,
      status: 'queued',
      isConcurrencySafe,  // 是否可以并发执行
      pendingProgress: [],
    })
    
    void this.processQueue()
  }
  
  // 判断是否可以执行工具
  private canExecuteTool(isConcurrencySafe: boolean) {
    const executing = this.tools.filter(t => t.status === 'executing')
    // 如果没有正在执行的，可以执行
    // 如果当前工具是并发安全的，且所有正在执行的也是并发安全的，可以执行
    return executing.length === 0 || 
           (isConcurrencySafe && executing.every(t => t.isConcurrencySafe))
  }
  
  // 获取剩余结果
  async *getRemainingResults(): AsyncGenerator<Message, void> {
    while (this.hasUnfinishedTools()) {
      // 处理队列
      await this.processQueue()
      
      // 输出已完成的结果
      for (const message of this.getCompletedResults()) {
        yield message
      }
      
      // 等待正在执行的工具
      if (this.hasExecutingTools() && !this.hasCompletedResults()) {
        await Promise.race([...executingPromises, progressPromise])
      }
    }
  }
}
```

## 五、消息流转

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           消息流转示意                                      │
│                                                                             │
│   时间线 ──────────────────────────────────────────────────────────────→   │
│                                                                             │
│   T1: 用户发送消息                                                          │
│   ┌─────────────────────────────────────────────────────────────────────┐  │
│   │ { type: "user", content: "重构 auth.ts" }                           │  │
│   └─────────────────────────────────────────────────────────────────────┘  │
│       │                                                                     │
│       ▼                                                                     │
│   T2: AI 响应，决定读取文件                                                 │
│   ┌─────────────────────────────────────────────────────────────────────┐  │
│   │ { type: "assistant", content: [                                     │  │
│   │   { type: "text", text: "让我先看看文件内容" },                      │  │
│   │   { type: "tool_use", name: "Read", input: { file_path: "..." } }   │  │
│   │ ]}                                                                   │  │
│   └─────────────────────────────────────────────────────────────────────┘  │
│       │                                                                     │
│       ▼                                                                     │
│   T3: 工具执行结果                                                          │
│   ┌─────────────────────────────────────────────────────────────────────┐  │
│   │ { type: "user", content: [                                          │  │
│   │   { type: "tool_result", tool_use_id: "...",                        │  │
│   │     content: "export function auth() { ... }" }                     │  │
│   │ ]}                                                                   │  │
│   └─────────────────────────────────────────────────────────────────────┘  │
│       │                                                                     │
│       ▼                                                                     │
│   T4: AI 响应，决定编辑文件                                                 │
│   ┌─────────────────────────────────────────────────────────────────────┐  │
│   │ { type: "assistant", content: [                                     │  │
│   │   { type: "text", text: "我来添加错误处理" },                        │  │
│   │   { type: "tool_use", name: "Edit", input: {                        │  │
│   │     file_path: "...", old_string: "...", new_string: "..."          │  │
│   │   }}                                                                 │  │
│   │ ]}                                                                   │  │
│   └─────────────────────────────────────────────────────────────────────┘  │
│       │                                                                     │
│       ▼                                                                     │
│   T5: 工具执行结果                                                          │
│   ┌─────────────────────────────────────────────────────────────────────┐  │
│   │ { type: "user", content: [                                          │  │
│   │   { type: "tool_result", tool_use_id: "...",                        │  │
│   │     content: "Successfully edited file" }                           │  │
│   │ ]}                                                                   │  │
│   └─────────────────────────────────────────────────────────────────────┘  │
│       │                                                                     │
│       ▼                                                                     │
│   T6: AI 最终响应，无工具调用                                               │
│   ┌─────────────────────────────────────────────────────────────────────┐  │
│   │ { type: "assistant", content: [                                     │  │
│   │   { type: "text", text: "重构完成！我添加了以下错误处理..." }        │  │
│   │ ]}                                                                   │  │
│   └─────────────────────────────────────────────────────────────────────┘  │
│                                                                             │
│   循环结束                                                                   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

## 六、与 Go 后端的类比

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                     Go 后端 vs Kode 对比                                    │
│                                                                             │
│   Go HTTP Handler                                                          │
│   ──────────────                                                            │
│   func HandleRequest(w http.ResponseWriter, r *http.Request) {              │
│       // 1. 解析请求                                                        │
│       // 2. 验证参数                                                        │
│       // 3. 调用服务                                                        │
│       // 4. 返回响应                                                        │
│       // 一次处理，一次返回                                                  │
│   }                                                                         │
│                                                                             │
│   Kode Agentic Loop                                                        │
│   ─────────────────                                                         │
│   async function* query(messages, ...) {                                    │
│       // 1. 处理消息                                                        │
│       // 2. 调用 AI                                                         │
│       // 3. 解析响应                                                        │
│       // 4. 如果有工具调用：执行工具，收集结果，回到步骤1                    │
│       // 5. 如果没有工具调用：返回最终响应                                  │
│       // 多轮循环，直到 AI 决定结束                                         │
│   }                                                                         │
│                                                                             │
│   核心差异：                                                                 │
│   ─────────                                                                 │
│   • Go: 确定性流程，程序员控制每一步                                        │
│   • Kode: AI 决策流程，模型控制每一步                                       │
│                                                                             │
│   • Go: 单次请求-响应                                                       │
│   • Kode: 循环直到完成                                                      │
│                                                                             │
│   • Go: 线性执行                                                            │
│   • Kode: 可能分支、回溯、重试                                              │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

## 七、关键设计点

### 7.1 异步生成器 (AsyncGenerator)

```typescript
// 使用 AsyncGenerator 的原因：
// 1. 支持流式输出 - 用户可以实时看到 AI 的响应
// 2. 支持中间状态 - 工具执行进度可以实时推送
// 3. 支持取消 - 用户可以随时中断

async function* query(...): AsyncGenerator<Message, void> {
  // 每次 yield 都会把控制权还给调用者
  yield message1  // 用户看到 AI 的思考
  yield message2  // 用户看到工具开始执行
  yield message3  // 用户看到工具执行结果
  // ...
}
```

### 7.2 递归调用

```typescript
// 递归调用的原因：
// AI 可能需要多轮工具调用才能完成任务
// 每轮调用都需要带上之前的上下文

yield* await queryCore(
  [...messages, assistantMessage, ...toolResults],  // 消息越来越长
  systemPrompt,
  context,
  canUseTool,
  updatedContext,  // 上下文可能被工具修改
)
```

### 7.3 权限检查

```typescript
// 每个工具执行前都要检查权限
const permissionResult = await canUseTool(
  tool,
  normalizedInput,
  context,
  assistantMessage,
)

if (permissionResult.result === false) {
  // 权限拒绝，返回错误消息
  yield createUserMessage([{
    type: 'tool_result',
    content: permissionResult.message,
    is_error: true,
    tool_use_id: toolUseID,
  }])
  return
}
```

### 7.4 Hook 系统

```typescript
// Hook 允许在关键点注入自定义逻辑

// 工具执行前
const hookOutcome = await runPreToolUseHooks({
  toolName: tool.name,
  toolInput: normalizedInput,
  // ...
})

if (hookOutcome.kind === 'block') {
  // Hook 阻止了工具执行
  yield createUserMessage([{
    type: 'tool_result',
    content: hookOutcome.message,
    is_error: true,
    tool_use_id: toolUseID,
  }])
  return
}

// 工具执行后
const postOutcome = await runPostToolUseHooks({
  toolName: tool.name,
  toolInput: normalizedInput,
  toolResult: result.data,
  // ...
})
```

## 八、下一步

建议继续阅读：

- **architecture-03-tools.zh-CN.md** - 工具系统的设计与实现
- **architecture-04-permissions.zh-CN.md** - 权限与安全治理
- **architecture-05-hooks.zh-CN.md** - Hook 系统详解
