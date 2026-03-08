# 工具系统详解

> 本文档详细解析 Kode 的工具系统设计，这是 AI Agent 能够"动手干活"的核心。

## 一、工具系统概览

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           工具系统架构                                      │
│                                                                             │
│   ┌─────────────────────────────────────────────────────────────────────┐  │
│   │                          AI 模型层                                  │  │
│   │                                                                     │  │
│   │   "我需要读取文件" → 决定调用 Read Tool                            │  │
│   │                                                                     │  │
│   └─────────────────────────────────────────────────────────────────────┘  │
│                                    │                                        │
│                                    ▼                                        │
│   ┌─────────────────────────────────────────────────────────────────────┐  │
│   │                          工具注册表                                  │  │
│   │                     src/core/tools/registry.ts                      │  │
│   │                                                                     │  │
│   │   ┌───────────┐ ┌───────────┐ ┌───────────┐ ┌───────────┐         │  │
│   │   │Read Tool  │ │Write Tool │ │Bash Tool  │ │Grep Tool  │ ...     │  │
│   │   └───────────┘ └───────────┘ └───────────┘ └───────────┘         │  │
│   │                                                                     │  │
│   └─────────────────────────────────────────────────────────────────────┘  │
│                                    │                                        │
│                                    ▼                                        │
│   ┌─────────────────────────────────────────────────────────────────────┐  │
│   │                          工具执行层                                  │  │
│   │                     src/core/tools/executor.ts                      │  │
│   │                                                                     │  │
│   │   1. 查找工具定义                                                   │  │
│   │   2. 验证输入参数                                                   │  │
│   │   3. 检查执行权限                                                   │  │
│   │   4. 执行工具逻辑                                                   │  │
│   │   5. 返回执行结果                                                   │  │
│   │                                                                     │  │
│   └─────────────────────────────────────────────────────────────────────┘  │
│                                    │                                        │
│                                    ▼                                        │
│   ┌─────────────────────────────────────────────────────────────────────┐  │
│   │                          外部世界                                    │  │
│   │                                                                     │  │
│   │   文件系统 │ Shell │ 网络 │ 数据库 │ 外部 API │ MCP Server         │  │
│   │                                                                     │  │
│   └─────────────────────────────────────────────────────────────────────┘  │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

## 二、Tool 接口定义

```typescript
// src/core/tools/tool.ts

export interface Tool<
  TInput extends z.ZodTypeAny = z.ZodTypeAny,
  TOutput = any,
> {
  // ========== 基本信息 ==========
  
  // 工具名称，AI 用这个名字来调用
  name: string
  
  // 工具描述，告诉 AI 这个工具是干什么的
  description?: string | ((input?: z.infer<TInput>) => Promise<string>)
  
  // ========== 输入定义 ==========
  
  // 输入参数的 Schema（使用 Zod 定义）
  inputSchema: TInput
  
  // 可选：直接提供 JSON Schema（某些场景需要）
  inputJSONSchema?: Record<string, unknown>
  
  // ========== 提示词生成 ==========
  
  // 生成给 AI 看的工具说明
  prompt: (options?: { safeMode?: boolean }) => Promise<string>
  
  // ========== 工具特性 ==========
  
  // 工具是否启用
  isEnabled: () => Promise<boolean>
  
  // 是否是只读工具（不修改外部状态）
  isReadOnly: (input?: z.infer<TInput>) => boolean
  
  // 是否可以并发执行
  isConcurrencySafe: (input?: z.infer<TInput>) => boolean
  
  // 是否需要权限检查
  needsPermissions: (input?: z.infer<TInput>) => boolean
  
  // 是否需要用户交互
  requiresUserInteraction?: (input?: z.infer<TInput>) => boolean
  
  // ========== 验证 ==========
  
  // 自定义输入验证
  validateInput?: (
    input: z.infer<TInput>,
    context?: ToolUseContext,
  ) => Promise<ValidationResult>
  
  // ========== 结果渲染 ==========
  
  // 将结果转换为 AI 可理解的文本
  renderResultForAssistant: (output: TOutput) => string | any[]
  
  // 将工具使用消息渲染给用户看
  renderToolUseMessage: (
    input: z.infer<TInput>,
    options: { verbose: boolean },
  ) => string | React.ReactElement | null
  
  // 工具被拒绝时的渲染
  renderToolUseRejectedMessage?: (...args: any[]) => React.ReactElement
  
  // 工具结果的渲染
  renderToolResultMessage?: (
    output: TOutput,
    options: { verbose: boolean },
  ) => React.ReactNode
  
  // ========== 核心执行 ==========
  
  // 工具的实际执行逻辑
  call: (
    input: z.infer<TInput>,
    context: ToolUseContext,
  ) => AsyncGenerator<
    | {
        type: 'result'
        data: TOutput
        resultForAssistant?: string | any[]
        newMessages?: unknown[]
        contextModifier?: {
          modifyContext: (ctx: ToolUseContext) => ToolUseContext
        }
      }
    | {
        type: 'progress'
        content: any
        normalizedMessages?: any[]
        tools?: any[]
      },
    void,
    unknown
  >
}
```

## 三、工具分类

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           工具分类                                          │
│                                                                             │
│   ┌─────────────────────────────────────────────────────────────────────┐  │
│   │                     文件系统工具 (filesystem/)                       │  │
│   │                                                                     │  │
│   │   ┌─────────────┐  ┌─────────────┐  ┌─────────────┐               │  │
│   │   │ FileReadTool│  │FileWriteTool│  │ FileEditTool│               │  │
│   │   │  读取文件   │  │  写入文件   │  │  编辑文件   │               │  │
│   │   └─────────────┘  └─────────────┘  └─────────────┘               │  │
│   │   ┌─────────────┐  ┌─────────────┐  ┌─────────────┐               │  │
│   │   │  GlobTool   │  │NotebookRead │  │NotebookEdit │               │  │
│   │   │  查找文件   │  │  读取笔记本  │  │  编辑笔记本  │               │  │
│   │   └─────────────┘  └─────────────┘  └─────────────┘               │  │
│   │                                                                     │  │
│   └─────────────────────────────────────────────────────────────────────┘  │
│                                                                             │
│   ┌─────────────────────────────────────────────────────────────────────┐  │
│   │                     系统工具 (system/)                               │  │
│   │                                                                     │  │
│   │   ┌─────────────┐  ┌─────────────┐  ┌─────────────┐               │  │
│   │   │  BashTool   │  │KillShellTool│  │TaskOutputTool│              │  │
│   │   │  执行命令   │  │  终止进程   │  │  获取任务输出│              │  │
│   │   └─────────────┘  └─────────────┘  └─────────────┘               │  │
│   │                                                                     │  │
│   └─────────────────────────────────────────────────────────────────────┘  │
│                                                                             │
│   ┌─────────────────────────────────────────────────────────────────────┐  │
│   │                     搜索工具 (search/)                               │  │
│   │                                                                     │  │
│   │   ┌─────────────┐  ┌─────────────┐                                │  │
│   │   │  GrepTool   │  │   LspTool   │                                │  │
│   │   │  文本搜索   │  │  LSP 查询   │                                │  │
│   │   └─────────────┘  └─────────────┘                                │  │
│   │                                                                     │  │
│   └─────────────────────────────────────────────────────────────────────┘  │
│                                                                             │
│   ┌─────────────────────────────────────────────────────────────────────┐  │
│   │                     网络工具 (network/)                              │  │
│   │                                                                     │  │
│   │   ┌─────────────┐  ┌─────────────┐                                │  │
│   │   │WebSearchTool│  │ WebFetchTool│                                │  │
│   │   │  网络搜索   │  │  获取网页   │                                │  │
│   │   └─────────────┘  └─────────────┘                                │  │
│   │                                                                     │  │
│   └─────────────────────────────────────────────────────────────────────┘  │
│                                                                             │
│   ┌─────────────────────────────────────────────────────────────────────┐  │
│   │                     交互工具 (interaction/)                          │  │
│   │                                                                     │  │
│   │   ┌─────────────┐  ┌─────────────┐  ┌─────────────┐               │  │
│   │   │AskUserTool  │  │ TodoWrite   │  │SlashCommand │               │  │
│   │   │  询问用户   │  │  任务管理   │  │  斜杠命令   │               │  │
│   │   └─────────────┘  └─────────────┘  └─────────────┘               │  │
│   │                                                                     │  │
│   └─────────────────────────────────────────────────────────────────────┘  │
│                                                                             │
│   ┌─────────────────────────────────────────────────────────────────────┐  │
│   │                     代理工具 (agent/)                                │  │
│   │                                                                     │  │
│   │   ┌─────────────┐  ┌─────────────┐  ┌─────────────┐               │  │
│   │   │  TaskTool   │  │EnterPlanMode│  │ExitPlanMode │               │  │
│   │   │  创建子任务 │  │  进入计划   │  │  退出计划   │               │  │
│   │   └─────────────┘  └─────────────┘  └─────────────┘               │  │
│   │                                                                     │  │
│   └─────────────────────────────────────────────────────────────────────┘  │
│                                                                             │
│   ┌─────────────────────────────────────────────────────────────────────┐  │
│   │                     AI 工具 (ai/)                                    │  │
│   │                                                                     │  │
│   │   ┌─────────────┐  ┌─────────────┐                                │  │
│   │   │ SkillTool   │  │AskExpertTool│                                │  │
│   │   │  调用技能   │  │  专家模型   │                                │  │
│   │   └─────────────┘  └─────────────┘                                │  │
│   │                                                                     │  │
│   └─────────────────────────────────────────────────────────────────────┘  │
│                                                                             │
│   ┌─────────────────────────────────────────────────────────────────────┐  │
│   │                     MCP 工具 (mcp/)                                  │  │
│   │                                                                     │  │
│   │   ┌─────────────┐  ┌─────────────┐  ┌─────────────┐               │  │
│   │   │  MCPTool    │  │ListMcpRes   │  │ReadMcpRes   │               │  │
│   │   │  调用 MCP   │  │  列出资源   │  │  读取资源   │               │  │
│   │   └─────────────┘  └─────────────┘  └─────────────┘               │  │
│   │                                                                     │  │
│   └─────────────────────────────────────────────────────────────────────┘  │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

## 四、工具示例：FileReadTool

```typescript
// src/tools/filesystem/FileReadTool/FileReadTool.ts (简化版)

import { z } from 'zod'
import { Tool } from '@tool'
import { readFile } from 'fs/promises'

// 1. 定义输入 Schema
const FileReadInputSchema = z.object({
  file_path: z.string().describe('The absolute path to the file to read'),
  offset: z.number().optional().describe('Line number to start reading from'),
  limit: z.number().optional().describe('Maximum number of lines to read'),
})

// 2. 定义输出类型
type FileReadOutput = {
  content: string
  path: string
  lines: number
}

// 3. 创建工具
export const FileReadTool: Tool<typeof FileReadInputSchema, FileReadOutput> = {
  // 基本信息
  name: 'Read',
  description: 'Reads a file from the local filesystem.',
  
  // 输入定义
  inputSchema: FileReadInputSchema,
  
  // 提示词
  prompt: async ({ safeMode }) => `
    Reads a file from the local filesystem.
    
    Usage notes:
    - Use absolute paths only
    - Can read text files, images, and PDFs
    ${safeMode ? '- Running in safe mode, some paths may be restricted' : ''}
  `,
  
  // 工具特性
  isEnabled: async () => true,
  isReadOnly: () => true,           // 只读，不修改文件
  isConcurrencySafe: () => true,    // 可以并发读取多个文件
  needsPermissions: () => true,     // 需要权限检查
  
  // 渲染结果给 AI
  renderResultForAssistant: (output) => output.content,
  
  // 渲染给用户看
  renderToolUseMessage: (input, { verbose }) => {
    if (verbose) {
      return `Reading file: ${input.file_path}`
    }
    return `Read(${input.file_path})`
  },
  
  // 核心执行逻辑
  call: async function* (input, context) {
    const { file_path, offset, limit } = input
    
    try {
      // 读取文件
      let content = await readFile(file_path, 'utf-8')
      
      // 处理 offset 和 limit
      if (offset || limit) {
        const lines = content.split('\n')
        const start = offset ?? 0
        const end = limit ? start + limit : lines.length
        content = lines.slice(start, end).join('\n')
      }
      
      // 返回结果
      yield {
        type: 'result',
        data: {
          content,
          path: file_path,
          lines: content.split('\n').length,
        },
      }
    } catch (error) {
      yield {
        type: 'result',
        data: {
          content: `Error reading file: ${error.message}`,
          path: file_path,
          lines: 0,
        },
      }
    }
  },
}
```

## 五、工具示例：BashTool

```typescript
// src/tools/system/BashTool/BashTool.ts (简化版)

import { z } from 'zod'
import { Tool } from '@tool'
import { BunShell } from '@utils/bun/shell'

const BashInputSchema = z.object({
  command: z.string().describe('The command to execute'),
  timeout: z.number().optional().describe('Timeout in seconds'),
  description: z.string().optional().describe('Why this command is being run'),
  run_in_background: z.boolean().optional(),
})

export const BashTool: Tool<typeof BashInputSchema, BashOutput> = {
  name: 'Bash',
  description: 'Executes a shell command',
  
  inputSchema: BashInputSchema,
  
  // Bash 命令可能修改系统，不是只读的
  isReadOnly: () => false,
  
  // 不同命令的并发安全性不同
  isConcurrencySafe: (input) => {
    const safeCommands = ['ls', 'cat', 'echo', 'git status', 'npm list']
    return safeCommands.some(cmd => input.command.startsWith(cmd))
  },
  
  // 总是需要权限检查
  needsPermissions: () => true,
  
  // 危险命令的验证
  validateInput: async (input, context) => {
    const dangerousPatterns = [
      /rm\s+-rf\s+\//,      // rm -rf /
      />\s*\/dev\/sd/,      // 写入磁盘
      /mkfs/,               // 格式化
      /dd\s+if=/,           // dd 命令
    ]
    
    for (const pattern of dangerousPatterns) {
      if (pattern.test(input.command)) {
        return {
          result: false,
          message: `Dangerous command detected: ${input.command}`,
        }
      }
    }
    
    return { result: true }
  },
  
  call: async function* (input, context) {
    const { command, timeout, run_in_background } = input
    
    // 更新状态
    setRequestStatus({ kind: 'tool', detail: 'Bash' })
    
    // 执行命令
    const shell = BunShell.getInstance()
    
    if (run_in_background) {
      // 后台执行
      const taskId = await shell.runBackground(command)
      yield {
        type: 'result',
        data: {
          stdout: '',
          stderr: '',
          exitCode: 0,
          background: true,
          taskId,
        },
        resultForAssistant: `Command started in background with ID: ${taskId}`,
      }
    } else {
      // 前台执行
      const result = await shell.run(command, { timeout })
      yield {
        type: 'result',
        data: result,
        resultForAssistant: result.stdout || result.stderr || '(no output)',
      }
    }
  },
}
```

## 六、工具注册与发现

```typescript
// src/tools/index.ts

import { Tool } from '@tool'
import { BashTool } from './system/BashTool/BashTool'
import { FileReadTool } from './filesystem/FileReadTool/FileReadTool'
import { FileWriteTool } from './filesystem/FileWriteTool/FileWriteTool'
import { FileEditTool } from './filesystem/FileEditTool/FileEditTool'
import { GlobTool } from './filesystem/GlobTool/GlobTool'
import { GrepTool } from './search/GrepTool/GrepTool'
import { WebSearchTool } from './network/WebSearchTool/WebSearchTool'
import { WebFetchTool } from './network/WebFetchTool/WebFetchTool'
import { TaskTool } from './agent/TaskTool/TaskTool'
import { AskUserQuestionTool } from './interaction/AskUserQuestionTool/AskUserQuestionTool'
import { TodoWriteTool } from './interaction/TodoWriteTool/TodoWriteTool'
import { SkillTool } from './ai/SkillTool/SkillTool'
import { MCPTool } from './mcp/MCPTool/MCPTool'
import { getMCPTools } from '@services/mcpClient'

// 获取所有内置工具
export const getAllTools = (): Tool[] => [
  TaskTool,
  BashTool,
  GlobTool,
  GrepTool,
  FileReadTool,
  FileEditTool,
  FileWriteTool,
  TodoWriteTool,
  WebSearchTool,
  WebFetchTool,
  AskUserQuestionTool,
  SkillTool,
  MCPTool,
  // ... 更多工具
]

// 获取启用的工具（包含 MCP 工具）
export const getTools = memoize(
  async (_includeOptional?: boolean): Promise<Tool[]> => {
    // 合并内置工具和 MCP 工具
    const tools = [...getAllTools(), ...(await getMCPTools())]
    
    // 过滤掉未启用的工具
    const isEnabled = await Promise.all(
      tools.map(tool => tool.isEnabled())
    )
    return tools.filter((_, i) => isEnabled[i])
  },
)

// 获取只读工具
export const getReadOnlyTools = memoize(async (): Promise<Tool[]> => {
  const tools = getAllTools().filter(tool => tool.isReadOnly())
  const isEnabled = await Promise.all(
    tools.map(tool => tool.isEnabled())
  )
  return tools.filter((_, index) => isEnabled[index])
})
```

## 七、工具执行流程

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        工具执行完整流程                                     │
│                                                                             │
│   AI 决定调用工具：Read({ file_path: "/src/app.ts" })                       │
│       │                                                                     │
│       ▼                                                                     │
│   ┌─────────────────────────────────────────────────────────────────────┐  │
│   │  1. 解析工具调用                                                    │  │
│   │     • 从 AI 响应中提取 tool_use block                               │  │
│   │     • 获取工具名称和输入参数                                        │  │
│   └─────────────────────────────────────────────────────────────────────┘  │
│       │                                                                     │
│       ▼                                                                     │
│   ┌─────────────────────────────────────────────────────────────────────┐  │
│   │  2. 查找工具定义                                                    │  │
│   │     • 在工具注册表中查找 name === "Read" 的工具                     │  │
│   │     • 如果找不到，返回 "No such tool" 错误                          │  │
│   └─────────────────────────────────────────────────────────────────────┘  │
│       │                                                                     │
│       ▼                                                                     │
│   ┌─────────────────────────────────────────────────────────────────────┐  │
│   │  3. 运行 PreToolUse Hooks                                           │  │
│   │     • 允许外部系统拦截或修改工具调用                                │  │
│   │     • 可以阻止执行、修改输入、添加警告                              │  │
│   └─────────────────────────────────────────────────────────────────────┘  │
│       │                                                                     │
│       ▼                                                                     │
│   ┌─────────────────────────────────────────────────────────────────────┐  │
│   │  4. 验证输入 Schema                                                 │  │
│   │     • 使用 Zod schema 验证输入参数                                  │  │
│   │     • 检查必填字段、类型、范围等                                    │  │
│   │     • 调用工具的 validateInput 方法                                 │  │
│   └─────────────────────────────────────────────────────────────────────┘  │
│       │                                                                     │
│       ▼                                                                     │
│   ┌─────────────────────────────────────────────────────────────────────┐  │
│   │  5. 检查权限                                                        │  │
│   │     • 调用 canUseTool(tool, input, context)                         │  │
│   │     • 检查文件路径权限                                              │  │
│   │     • 检查命令权限                                                  │  │
│   │     • 根据权限模式决定：允许/拒绝/询问用户                          │  │
│   └─────────────────────────────────────────────────────────────────────┘  │
│       │                                                                     │
│       ├─────────────────────────────────────────┐                          │
│       │                                         │                          │
│       ▼ 权限通过                                ▼ 权限拒绝                  │
│   ┌─────────────────────┐              ┌─────────────────────┐            │
│   │  6. 执行工具        │              │  返回权限错误       │            │
│   │     tool.call()     │              │  tool_result:       │            │
│   │                     │              │  is_error: true     │            │
│   └──────────┬──────────┘              └─────────────────────┘            │
│              │                                                              │
│              ▼                                                              │
│   ┌─────────────────────────────────────────────────────────────────────┐  │
│   │  7. 收集执行结果                                                    │  │
│   │     • 工具通过 yield 返回结果                                       │  │
│   │     • 可以返回多个 progress 消息                                    │  │
│   │     • 最后返回一个 result                                           │  │
│   └─────────────────────────────────────────────────────────────────────┘  │
│       │                                                                     │
│       ▼                                                                     │
│   ┌─────────────────────────────────────────────────────────────────────┐  │
│   │  8. 运行 PostToolUse Hooks                                          │  │
│   │     • 允许外部系统处理工具结果                                      │  │
│   │     • 可以添加警告、系统消息                                        │  │
│   └─────────────────────────────────────────────────────────────────────┘  │
│       │                                                                     │
│       ▼                                                                     │
│   ┌─────────────────────────────────────────────────────────────────────┐  │
│   │  9. 返回工具结果给 AI                                               │  │
│   │     • 格式化为 tool_result 消息                                     │  │
│   │     • AI 在下一轮对话中可以看到这个结果                             │  │
│   └─────────────────────────────────────────────────────────────────────┘  │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

## 八、工具与 AI 模型的交互

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                     工具 Schema 如何传给 AI                                 │
│                                                                             │
│   1. 工具定义                                                              │
│   ┌─────────────────────────────────────────────────────────────────────┐  │
│   │   const ReadTool = {                                                │  │
│   │     name: 'Read',                                                   │  │
│   │     inputSchema: z.object({                                         │  │
│   │       file_path: z.string(),                                        │  │
│   │       offset: z.number().optional(),                                │  │
│   │     }),                                                             │  │
│   │   }                                                                 │  │
│   └─────────────────────────────────────────────────────────────────────┘  │
│                                                                             │
│   2. 转换为 API 格式                                                       │
│   ┌─────────────────────────────────────────────────────────────────────┐  │
│   │   {                                                                 │  │
│   │     "name": "Read",                                                 │  │
│   │     "description": "Reads a file from the local filesystem...",     │  │
│   │     "input_schema": {                                               │  │
│   │       "type": "object",                                             │  │
│   │       "properties": {                                               │  │
│   │         "file_path": {                                              │  │
│   │           "type": "string",                                         │  │
│   │           "description": "The absolute path to the file..."         │  │
│   │         },                                                          │  │
│   │         "offset": {                                                 │  │
│   │           "type": "number",                                         │  │
│   │           "description": "Line number to start reading from"        │  │
│   │         }                                                           │  │
│   │       },                                                            │  │
│   │       "required": ["file_path"]                                     │  │
│   │     }                                                               │  │
│   │   }                                                                 │  │
│   └─────────────────────────────────────────────────────────────────────┘  │
│                                                                             │
│   3. AI 根据 Schema 生成调用                                               │
│   ┌─────────────────────────────────────────────────────────────────────┐  │
│   │   {                                                                 │  │
│   │     "type": "tool_use",                                             │  │
│   │     "id": "toolu_123",                                              │  │
│   │     "name": "Read",                                                 │  │
│   │     "input": {                                                      │  │
│   │       "file_path": "/src/app.ts"                                    │  │
│   │     }                                                               │  │
│   │   }                                                                 │  │
│   └─────────────────────────────────────────────────────────────────────┘  │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

## 九、MCP 工具集成

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        MCP 工具集成                                         │
│                                                                             │
│   MCP (Model Context Protocol) 是一种让 AI 模型调用外部工具的协议           │
│                                                                             │
│   ┌─────────────────────────────────────────────────────────────────────┐  │
│   │                                                                     │  │
│   │   ┌───────────────┐         ┌───────────────┐                      │  │
│   │   │   Kode CLI    │         │  MCP Server   │                      │  │
│   │   │               │  JSON-RPC│               │                      │  │
│   │   │  MCP Client   │◄───────►│  (外部服务)    │                      │  │
│   │   │               │         │               │                      │  │
│   │   └───────────────┘         └───────────────┘                      │  │
│   │          │                           │                              │  │
│   │          │                           │                              │  │
│   │          ▼                           ▼                              │  │
│   │   ┌───────────────┐         ┌───────────────┐                      │  │
│   │   │ MCPTool       │         │ 工具实现      │                      │  │
│   │   │ (统一调用入口) │         │ • 数据库查询  │                      │  │
│   │   │               │         │ • API 调用    │                      │  │
│   │   └───────────────┘         │ • 文件操作    │                      │  │
│   │                             │ • 任何服务    │                      │  │
│   │                             └───────────────┘                      │  │
│   │                                                                     │  │
│   └─────────────────────────────────────────────────────────────────────┘  │
│                                                                             │
│   MCP 工具命名规则：mcp__<server_name>__<tool_name>                        │
│   例如：mcp__github__create_issue                                          │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

## 十、如何创建新工具

```typescript
// 1. 创建工具文件 src/tools/my/MyTool.ts

import { z } from 'zod'
import { Tool } from '@tool'

const MyInputSchema = z.object({
  param1: z.string().describe('参数1的说明'),
  param2: z.number().optional().describe('可选参数2'),
})

type MyOutput = {
  result: string
}

export const MyTool: Tool<typeof MyInputSchema, MyOutput> = {
  name: 'MyTool',
  description: '这是一个自定义工具',
  inputSchema: MyInputSchema,
  
  prompt: async () => '这个工具用来做某事...',
  
  isEnabled: async () => true,
  isReadOnly: () => true,
  isConcurrencySafe: () => true,
  needsPermissions: () => false,
  
  renderResultForAssistant: (output) => output.result,
  renderToolUseMessage: (input) => `MyTool(${input.param1})`,
  
  call: async function* (input, context) {
    // 实现工具逻辑
    const result = doSomething(input)
    
    yield {
      type: 'result',
      data: { result },
    }
  },
}

// 2. 注册到 src/tools/index.ts

import { MyTool } from './my/MyTool'

export const getAllTools = (): Tool[] => [
  // ... 其他工具
  MyTool,
]
```

## 十一、下一步

建议继续阅读：

- **architecture-04-permissions.zh-CN.md** - 权限与安全治理
- **architecture-06-extensions.zh-CN.md** - MCP/Plugins/Skills 扩展
