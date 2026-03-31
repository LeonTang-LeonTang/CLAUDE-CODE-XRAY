# Tool Usage: The Execution Layer

Tools are the "hands" of Claude Code — they transform AI decisions into real actions. This document explains the complete tool system architecture, from tool definitions to execution pipelines.

## Tool Architecture Overview

```
┌─────────────────────────────────────────────────────────────┐
│                    TOOL SYSTEM                               │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │              TOOL REGISTRY (tools.ts)                │   │
│  │  - getTools() → all available tools                │   │
│  │  - findToolByName(name) → tool instance            │   │
│  │  - Tool permission requirements                    │   │
│  └─────────────────────────────────────────────────────┘   │
│                           │                                │
│  ┌────────────────────────▼────────────────────────────┐   │
│  │              TOOL DEFINITION (Tool.ts)              │   │
│  │  - name, description, input schema                   │   │
│  │  - permission level                                 │   │
│  │  - progress type                                    │   │
│  │  - execute() method                                 │   │
│  └─────────────────────────────────────────────────────┘   │
│                           │                                │
│  ┌────────────────────────▼────────────────────────────┐   │
│  │          TOOL ORCHESTRATION (toolOrchestration.ts)   │   │
│  │  - Partition by concurrency safety                   │   │
│  │  - Concurrent execution for read-only                  │   │
│  │  - Serial execution for stateful                      │   │
│  └─────────────────────────────────────────────────────┘   │
│                           │                                │
│  ┌────────────────────────▼────────────────────────────┐   │
│  │            TOOL EXECUTION (toolExecution.ts)          │   │
│  │  - Permission checking                               │   │
│  │  - Progress streaming                                │   │
│  │  - Error handling                                    │   │
│  └─────────────────────────────────────────────────────┘   │
│                           │                                │
│  ┌────────────────────────▼────────────────────────────┐   │
│  │           TOOL IMPLEMENTATIONS (46+ tools)          │   │
│  │  - BashTool, FileReadTool, WebSearchTool, etc.     │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

## Tool Definition (Tool.ts)

### The Tool Interface

Every tool implements this interface:

```typescript
// Tool.ts
export interface Tool {
  // Unique identifier
  name: string
  
  // Human-readable description (shown to AI)
  description: string
  
  // JSON Schema for input validation
  inputSchema: ToolInputJSONSchema
  
  // Permission requirements
  permission?: ToolPermission
  
  // Progress streaming type
  progress?: ToolProgressType
  
  // The actual execution
  execute(
    input: unknown,
    context: ToolUseContext
  ): AsyncGenerator<ToolProgress> | Promise<ToolResult>
}

export interface ToolUseContext {
  canUseTool: CanUseToolFn
  getAppState: () => AppState
  setAppState: SetAppState
  abortController: AbortController
  stream?: WritableStream
}
```

### Tool Metadata

```typescript
// Example tool definition
export const BASH_TOOL: Tool = {
  name: 'BashTool',
  description: 'Execute a bash command in the terminal',
  inputSchema: {
    type: 'object',
    properties: {
      command: {
        type: 'string',
        description: 'The bash command to execute',
      },
      timeout: {
        type: 'number',
        description: 'Timeout in milliseconds (default: 60000)',
      },
      workingDirectory: {
        type: 'string',
        description: 'Directory to run command in',
      },
    },
    required: ['command'],
  },
  permission: {
    level: 'high',
    category: 'shell',
    dangerous: true,
  },
  progress: 'bash',
}
```

## Tool Registry (tools.ts)

### Registration

```typescript
// tools.ts - All available tools
import { BashTool } from './tools/BashTool/BashTool.js'
import { FileReadTool } from './tools/FileReadTool/FileReadTool.js'
import { FileEditTool } from './tools/FileEditTool/FileEditTool.js'
// ... 40+ more tools

export function getTools(): Tool[] {
  return [
    // Shell tools
    BashTool,
    PowerShellTool,
    
    // File tools
    FileReadTool,
    FileEditTool,
    FileWriteTool,
    GlobTool,
    GrepTool,
    
    // Web tools
    WebSearchTool,
    WebFetchTool,
    
    // Agent tools
    AgentTool,
    TaskCreateTool,
    TaskGetTool,
    TaskUpdateTool,
    TaskStopTool,
    
    // MCP tools (added dynamically)
    ...getMCPTools(),
    
    // More...
  ]
}
```

### Tool Lookup

```typescript
// Find tool by name
export function findToolByName(name: string): Tool | undefined {
  return getTools().find(tool => tool.name === name)
}

// Check if tool matches name (handles aliases)
export function toolMatchesName(tool: Tool, name: string): boolean {
  return tool.name === name ||
         tool.aliases?.includes(name) ||
         false
}
```

## Tool Orchestration (toolOrchestration.ts)

### The Orchestration Pipeline

```typescript
// toolOrchestration.ts - orchestrates multiple tool calls
export async function* runTools(
  toolUseMessages: ToolUseBlock[],
  assistantMessages: AssistantMessage[],
  canUseTool: CanUseToolFn,
  toolUseContext: ToolUseContext,
): AsyncGenerator<MessageUpdate, void> {
  // 1. Partition tools by concurrency safety
  const { safeTools, unsafeTools } = partitionBySafety(toolUseMessages)
  
  // 2. Run read-only tools concurrently
  if (safeTools.length > 0) {
    yield* runToolsConcurrently(safeTools, toolUseContext)
  }
  
  // 3. Run stateful tools serially
  if (unsafeTools.length > 0) {
    yield* runToolsSerially(unsafeTools, toolUseContext)
  }
}
```

### Concurrency Safety Partitioning

```typescript
// Which tools can run concurrently?
const READ_ONLY_TOOLS = new Set([
  'GlobTool',
  'GrepTool',
  'FileReadTool',
  'WebFetchTool',
  'WebSearchTool',
  'MCPTool',  // Depends on specific tool
])

const STATEFUL_TOOLS = new Set([
  'BashTool',
  'FileWriteTool',
  'FileEditTool',
  'NotebookEditTool',
  'AgentTool',  // Creates sub-agents
])

function isConcurrencySafe(toolName: string): boolean {
  // Check if all dependencies are read-only
  const tool = findToolByName(toolName)
  if (!tool) return false
  
  return READ_ONLY_TOOLS.has(tool.name)
}

function partitionBySafety(
  toolUses: ToolUseBlock[]
): { safeTools: ToolUseBlock[], unsafeTools: ToolUseBlock[] } {
  return {
    safeTools: toolUses.filter(t => isConcurrencySafe(t.name)),
    unsafeTools: toolUses.filter(t => !isConcurrencySafe(t.name)),
  }
}
```

### Concurrent Execution

```typescript
// Run read-only tools in parallel
async function* runToolsConcurrently(
  tools: ToolUseBlock[],
  context: ToolUseContext,
): AsyncGenerator<MessageUpdate> {
  // Use Promise.allSettled to handle failures gracefully
  const promises = tools.map(async (toolUse) => {
    try {
      return {
        success: true,
        result: await executeSingleTool(toolUse, context),
      }
    } catch (error) {
      return {
        success: false,
        error,
        toolUse,
      }
    }
  })
  
  const results = await Promise.all(promises)
  
  // Yield results in order
  for (const result of results) {
    if (result.success) {
      yield { message: result.result }
    } else {
      yield { message: createErrorMessage(result.error) }
    }
  }
}
```

## Tool Execution (toolExecution.ts)

### Single Tool Execution

```typescript
// toolExecution.ts - executes a single tool call
export async function* runToolUse(
  toolUse: ToolUseBlock,
  context: ToolUseContext,
): AsyncGenerator<MessageUpdate, void> {
  // 1. Find the tool
  const tool = findToolByName(toolUse.name)
  if (!tool) {
    yield createErrorMessage(`Tool not found: ${toolUse.name}`)
    return
  }
  
  // 2. Check permissions
  const canUse = await context.canUseTool(tool.name)
  if (!canUse) {
    yield* requestPermission(tool, context)
    return
  }
  
  // 3. Validate input
  const validation = validateInput(tool, toolUse.input)
  if (!validation.valid) {
    yield createErrorMessage(`Invalid input: ${validation.error}`)
    return
  }
  
  // 4. Execute with progress streaming
  try {
    const result = yield* executeWithProgress(tool, validation.input, context)
    yield { message: createSuccessMessage(result) }
  } catch (error) {
    yield { message: createErrorMessage(error) }
  }
}
```

### Permission Flow

```typescript
// Permission request flow
async function* requestPermission(
  tool: Tool,
  context: ToolUseContext,
): AsyncGenerator<MessageUpdate> {
  // 1. Update state to show permission request
  context.setAppState((s) => ({
    ...s,
    permissionState: {
      status: 'requesting',
      toolName: tool.name,
      reason: tool.permission?.reason,
    },
  }))
  
  // 2. Send request to user
  yield {
    message: createPermissionRequestMessage(tool),
  }
  
  // 3. Wait for response (blocking)
  const response = await waitForPermissionResponse()
  
  // 4. Handle response
  if (response.approved) {
    if (response.remember) {
      await rememberPermission(tool.name)
    }
  } else {
    yield {
      message: createPermissionDeniedMessage(tool.name, response.reason),
    }
  }
}

async function waitForPermissionResponse(): Promise<PermissionResponse> {
  return new Promise((resolve) => {
    // This would be resolved by the UI when user clicks approve/deny
    permissionResolver = resolve
  })
}
```

## Individual Tool Implementations

### BashTool (Shell Execution)

The most complex tool, executing shell commands:

```typescript
// tools/BashTool/BashTool.ts
export class BashTool implements Tool {
  name = 'BashTool'
  description = 'Execute a bash command'
  
  async *execute(
    input: BashToolInput,
    context: ToolUseContext
  ): AsyncGenerator<BashProgress, BashResult> {
    const { command, timeout = 60000 } = input
    
    // 1. Security checks
    yield { type: 'security_check' }
    const securityResult = await checkCommandSecurity(command)
    if (!securityResult.safe) {
      return {
        success: false,
        error: securityResult.reason,
      }
    }
    
    // 2. Execute command
    yield { type: 'executing', command }
    const result = await spawnProcess(command, {
      cwd: context.cwd,
      timeout,
      env: process.env,
    })
    
    // 3. Stream output
    for await (const chunk of result.stdout) {
      yield { type: 'stdout', data: chunk }
    }
    
    for await (const chunk of result.stderr) {
      yield { type: 'stderr', data: chunk }
    }
    
    // 4. Return result
    return {
      success: result.exitCode === 0,
      stdout: result.stdoutText,
      stderr: result.stderrText,
      exitCode: result.exitCode,
    }
  }
}
```

### FileReadTool (File Reading)

```typescript
// tools/FileReadTool/FileReadTool.ts
export class FileReadTool implements Tool {
  name = 'FileReadTool'
  description = 'Read contents of a file'
  
  async execute(
    input: FileReadInput,
    context: ToolUseContext
  ): Promise<FileReadResult> {
    const { path, startLine, endLine, limit } = input
    
    // 1. Resolve path (prevent directory traversal)
    const resolvedPath = resolvePath(path, context.cwd)
    
    // 2. Check if file exists
    if (!await exists(resolvedPath)) {
      return { success: false, error: 'File not found' }
    }
    
    // 3. Read file
    let content = await readFile(resolvedPath, 'utf-8')
    
    // 4. Handle line range
    if (startLine || endLine) {
      const lines = content.split('\n')
      const start = (startLine ?? 1) - 1
      const end = endLine ?? lines.length
      content = lines.slice(start, end).join('\n')
    }
    
    // 5. Apply limit
    if (limit && content.length > limit) {
      content = content.slice(0, limit) + '\n[...truncated...]'
    }
    
    return { success: true, content }
  }
}
```

### FileEditTool (File Editing)

```typescript
// tools/FileEditTool/FileEditTool.ts
export class FileEditTool implements Tool {
  name = 'FileEditTool'
  description = 'Make changes to a file'
  
  async execute(
    input: FileEditInput,
    context: ToolUseContext
  ): Promise<FileEditResult> {
    const { path, edits } = input
    
    // 1. Read current file
    const currentContent = await readFile(path, 'utf-8')
    
    // 2. Apply edits
    let newContent = currentContent
    for (const edit of edits) {
      newContent = applyEdit(newContent, edit)
    }
    
    // 3. Write back
    await writeFile(path, newContent)
    
    return {
      success: true,
      newContent,
      diff: computeDiff(currentContent, newContent),
    }
  }
}

// Apply a single edit
function applyEdit(content: string, edit: Edit): string {
  switch (edit.type) {
    case 'replace':
      return content.slice(0, edit.start) + 
             edit.newText + 
             content.slice(edit.end)
             
    case 'insert':
      const lines = content.split('\n')
      lines.splice(edit.line - 1, 0, edit.newText)
      return lines.join('\n')
      
    case 'delete':
      const delLines = content.split('\n')
      delLines.splice(edit.startLine - 1, edit.endLine - edit.startLine + 1)
      return delLines.join('\n')
      
    default:
      throw new Error(`Unknown edit type: ${edit.type}`)
  }
}
```

### MCPTool (Dynamic Tool Wrapper)

```typescript
// tools/MCPTool/MCPTool.ts
export class MCPTool implements Tool {
  name = 'MCPTool'
  description = 'Tool from MCP server'
  
  async execute(
    input: MCPToolInput,
    context: ToolUseContext
  ): Promise<MCPToolResult> {
    const { serverName, toolName, arguments: args } = input
    
    // 1. Get MCP client
    const client = getMCPClient(serverName)
    
    // 2. Call the tool
    const result = await client.callTool(toolName, args)
    
    // 3. Format result
    return {
      success: true,
      content: formatMCPResult(result),
    }
  }
}
```

## Tool Progress Streaming

### Progress Types

```typescript
// ToolProgress type varies by tool
type ToolProgress =
  | BashProgress
  | FileReadProgress
  | WebSearchProgress
  | AgentProgress
  | MCPToolProgress

// Example: BashTool progress
interface BashProgress {
  type: 'stdout' | 'stderr' | 'exit' | 'error'
  data?: string
  exitCode?: number
  error?: string
}
```

### Progress Streaming in UI

```typescript
// UI receives progress updates
function handleToolProgress(progress: ToolProgress) {
  switch (progress.type) {
    case 'stdout':
      appendToOutput(progress.data)
      break
    case 'stderr':
      appendToError(progress.data)
      break
    case 'exit':
      showExitCode(progress.exitCode)
      break
    case 'error':
      showError(progress.error)
      break
  }
}
```

## Error Handling

### Tool Error Types

```typescript
// Error handling in tool execution
export class ToolExecutionError extends Error {
  constructor(
    message: string,
    public toolName: string,
    public toolInput: unknown,
    public originalError?: Error
  ) {
    super(message)
  }
}

// Common error types:
const TOOL_ERRORS = {
  NOT_FOUND: 'Tool not found',
  PERMISSION_DENIED: 'Permission denied',
  INVALID_INPUT: 'Invalid tool input',
  TIMEOUT: 'Tool execution timed out',
  FILE_NOT_FOUND: 'File not found',
  NETWORK_ERROR: 'Network error',
  EXECUTION_FAILED: 'Tool execution failed',
}
```

### Error Recovery

```typescript
// Retry logic for transient failures
async function executeWithRetry(
  tool: Tool,
  input: unknown,
  context: ToolUseContext,
  maxRetries: number = 2
): Promise<ToolResult> {
  for (let attempt = 0; attempt <= maxRetries; attempt++) {
    try {
      return await tool.execute(input, context)
    } catch (error) {
      if (attempt === maxRetries || !isRetryableError(error)) {
        throw error
      }
      // Exponential backoff
      await sleep(Math.pow(2, attempt) * 1000)
    }
  }
}

function isRetryableError(error: Error): boolean {
  return error instanceof NetworkError ||
         error instanceof TimeoutError ||
         false
}
```

## Security Considerations

### BashTool Security

```typescript
// tools/BashTool/bashSecurity.ts

// Commands that are always dangerous
const BLOCKED_COMMANDS = [
  ':(){:|:&};:',    // Fork bomb
  'rm -rf /',
  'dd if=/dev/zero of=/dev/sda',
]

// Commands needing extra confirmation
const DANGEROUS_COMMANDS = [
  /^rm\s+-rf/,
  /^mkfs\./,
  /^dd\s+if=.*of=\/dev\//,
  /^>:?\s*\/dev\/null/,
  /^curl\s+.*\|\s*sh/,
]

export function checkCommandSecurity(command: string): SecurityCheckResult {
  // Check for blocked commands
  for (const blocked of BLOCKED_COMMANDS) {
    if (command.includes(blocked)) {
      return { safe: false, reason: 'Blocked command' }
    }
  }
  
  // Check for dangerous patterns
  for (const dangerous of DANGEROUS_COMMANDS) {
    if (dangerous.test(command)) {
      return { 
        safe: false, 
        reason: 'Dangerous command - requires extra confirmation' 
      }
    }
  }
  
  return { safe: true }
}
```

## Summary

The tool system demonstrates several key patterns:

1. **Interface-based design** — All tools implement the same interface
2. **Pipeline architecture** — Orchestration → Execution → Implementation
3. **Progress streaming** — Real-time feedback for long operations
4. **Permission gates** — Safety checks before execution
5. **Concurrency partitioning** — Safe parallelization
6. **Security-first** — Especially for shell execution
7. **Error recovery** — Retry logic for transient failures
