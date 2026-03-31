# Abstractions: Key Interfaces and Types

This document details the key abstractions in Claude Code — the interfaces, types, and contracts that define how components interact.

## Core Abstractions

### 1. Tool Abstraction

The most important abstraction in Claude Code:

```typescript
// Tool.ts
export interface Tool {
  name: string
  description: string
  inputSchema: ToolInputJSONSchema
  permission?: ToolPermission
  progress?: ToolProgressType
}

export type ToolProgressType = 
  | 'bash'
  | 'file'
  | 'web'
  | 'mcp'
  | 'agent'

// Tools implement this interface
export interface ExecutableTool extends Tool {
  execute(
    input: unknown,
    context: ToolUseContext
  ): AsyncGenerator<ToolProgress> | Promise<ToolResult>
}

// Tool use from API response
export interface ToolUseBlock {
  id: string
  name: string
  input: unknown
}

// Tool result for API
export interface ToolResult {
  content: string
  isError?: boolean
}
```

**Why Important**: All tools are interchangeable. The query engine doesn't know or care if it's a BashTool or FileReadTool — it just calls `execute()`.

---

### 2. Message Abstraction

```typescript
// types/message.ts
export type Message =
  | UserMessage
  | AssistantMessage
  | SystemMessage
  | ToolResultMessage
  | ProgressMessage
  | HookResultMessage

export interface BaseMessage {
  type: string
  timestamp: Date
}

export interface UserMessage extends BaseMessage {
  type: 'user'
  content: string | ContentBlock[]
  attachments?: Attachment[]
}

export interface AssistantMessage extends BaseMessage {
  type: 'assistant'
  content: string | ContentBlock[]
  toolCalls?: ToolCall[]
  usage?: Usage
  stopReason?: string
}

export interface ToolResultMessage extends BaseMessage {
  type: 'tool_result'
  toolUseId: string
  toolName: string
  content: string
  isError?: boolean
}

// API-compatible message format
export interface APIMessage {
  role: 'user' | 'assistant' | 'system'
  content: string | ContentBlock[]
}
```

**Why Important**: Claude Code uses a richer internal format than the API, requiring normalization.

---

### 3. Context Abstraction

```typescript
// ToolUseContext - passed to all tools
export interface ToolUseContext {
  canUseTool: CanUseToolFn
  getAppState: () => AppState
  setAppState: SetAppState
  abortController: AbortController
  cwd: string
  mcpClients: Map<string, MCPClient>
  tokenBudget: TokenBudget
}

export type CanUseToolFn = (toolName: string) => Promise<boolean>

export interface TokenBudget {
  remaining: number
  total: number
  isNearLimit: boolean
}
```

**Why Important**: Tools don't access global state directly — they receive everything they need through context. This makes tools testable and flexible.

---

### 4. State Abstraction

```typescript
// AppState - global application state
export interface AppState {
  // Conversation
  messages: Message[]
  
  // Tasks (background operations)
  tasks: TaskState[]
  
  // Permissions
  permissions: PermissionState
  permissionHistory: PermissionRecord[]
  
  // Multi-agent
  teammates: Teammate[]
  
  // Session
  sessionId: string
  sessionStarted: Date
  
  // UI
  isLoading: boolean
  activeTask: string | null
  inputValue: string
  
  // Settings
  settings: Settings
  
  // MCP
  mcpServers: MCPServerState[]
}

// State store interface
export type Store<T> = {
  getState: () => T
  setState: (updater: (prev: T) => T) => void
  subscribe: (listener: () => void) => () => void
}
```

---

### 5. Stream Event Abstraction

```typescript
// QueryEngine streams these events
export type StreamEvent =
  | { type: 'text'; text: string }
  | { type: 'tool_start'; tool: ToolUseBlock }
  | { type: 'tool_input'; toolUseId: string; chunk: string }
  | { type: 'tool_complete'; tool: ToolUseBlock }
  | { type: 'tool_error'; toolUseId: string; error: string }
  | { type: 'complete'; stopReason: string; usage: Usage }
  | { type: 'error'; error: APIError }
  | { type: 'progress'; progress: ToolProgress }

// Tool progress types
export type ToolProgress =
  | BashProgress
  | FileProgress
  | WebProgress
  | AgentProgress

export interface BashProgress {
  type: 'bash'
  stage: 'preparing' | 'running' | 'complete'
  stdout?: string
  stderr?: string
  exitCode?: number
}
```

---

### 6. Query Abstraction

```typescript
// query.ts
export interface QueryParams {
  input: string | UserMessage
  model?: string
  tools?: Tool[]
  systemPrompt?: string
  attachments?: Attachment[]
  agentId?: string
}

export interface QueryResult {
  message: AssistantMessage
  usage: Usage
  stopReason: string
}

export type QueryEvent = StreamEvent
```

---

### 7. Task Abstraction

```typescript
// Task.ts
export interface Task {
  type: TaskType
  name: string
  kill(taskId: string, setAppState: SetAppState): Promise<void>
}

export type TaskType = 
  | 'local_bash'
  | 'local_agent'
  | 'remote_agent'
  | 'in_process_teammate'
  | 'dream'

export interface TaskState {
  id: string
  type: TaskType
  status: TaskStatus
  description: string
  startTime: number
  outputFile: string
}

export type TaskStatus = 
  | 'pending'
  | 'running'
  | 'completed'
  | 'failed'
  | 'killed'
```

---

### 8. MCP Abstraction

```typescript
// services/mcp/types.ts
export interface MCPServerConfig {
  name: string
  command?: string[]
  args?: string[]
  env?: Record<string, string>
  url?: string
  transport?: 'stdio' | 'sse' | 'http'
}

export interface MCPServerConnection {
  name: string
  client: Client
  status: 'connecting' | 'connected' | 'error'
  tools: McpTool[]
  resources: McpResource[]
}

export interface McpTool {
  name: string
  description: string
  inputSchema: object
}

export interface McpResource {
  uri: string
  name: string
  mimeType?: string
}
```

---

### 9. Permission Abstraction

```typescript
// types/permissions.ts
export interface ToolPermission {
  level: 'none' | 'low' | 'medium' | 'high' | 'critical'
  category: 'read' | 'write' | 'execute' | 'network' | 'system'
  dangerous?: boolean
}

export type PermissionState =
  | { status: 'ready' }
  | { status: 'requesting'; toolName: string; reason: string }
  | { status: 'approved'; toolName: string; remember: boolean }
  | { status: 'denied'; toolName: string; reason?: string }

export interface PermissionRecord {
  toolName: string
  decision: 'approved' | 'denied'
  timestamp: Date
  reason?: string
}
```

---

### 10. Hook Abstraction

```typescript
// types/hooks.ts
export interface Hook {
  name: string
  trigger: HookTrigger
  priority: number
  execute(context: HookContext): Promise<HookResult>
}

export type HookTrigger =
  | 'before_query'
  | 'after_query'
  | 'before_tool'
  | 'after_tool'
  | 'on_error'

export interface HookContext {
  // Varies by trigger type
}

export interface HookResult {
  blocked?: boolean
  message?: string
  modified?: Partial<any>
}
```

---

## Interface Contracts

### Tool Contract

```typescript
// What tools MUST provide
interface ToolContract {
  // Identity
  name: string                    // Unique identifier
  description: string            // Shown to AI model
  
  // Schema
  inputSchema: JSONSchema        // Validates input
  
  // Behavior (one of these)
  execute(
    input: unknown,               // Validated against schema
    context: ToolUseContext       // Everything tool needs
  ): 
    | Promise<ToolResult>         // Simple result
    | AsyncGenerator<Progress>    // Streaming result
}

// What tools RECEIVE
interface ToolUseContext {
  // Permission
  canUseTool: (name: string) => Promise<boolean>
  
  // State
  getAppState: () => AppState
  setAppState: (updater: (s: AppState) => AppState) => void
  
  // Execution
  abortController: AbortController
  
  // Environment
  cwd: string
}

// What tools MUST return
interface ToolResult {
  content: string                  // Result content
  isError?: boolean               // Error flag
}
```

### Store Contract

```typescript
// What the store provides
interface StoreContract<T> {
  // Read
  getState(): T                   // Synchronous read
  
  // Write
  setState(updater: (prev: T) => T): void
  
  // Subscribe
  subscribe(listener: () => void): () => void
}

// Invariants:
// 1. setState always notifies if state changed
// 2. getState always returns latest state
// 3. subscribe returns unsubscribe function
```

---

## Abstract Base Classes

### BashTool Base

```typescript
// Common interface for shell tools
abstract class BaseShellTool implements ExecutableTool {
  abstract name: string
  abstract description: string
  
  async *execute(
    input: ShellInput,
    context: ToolUseContext
  ): AsyncGenerator<ShellProgress> {
    // 1. Validate input
    const validated = this.validate(input)
    
    // 2. Security check
    const security = await this.checkSecurity(input.command)
    if (!security.safe) {
      return { type: 'error', message: security.reason }
    }
    
    // 3. Execute
    const result = yield* this.runCommand(input, context)
    
    // 4. Format result
    return this.formatResult(result)
  }
  
  protected abstract validate(input: unknown): ShellInput
  protected abstract checkSecurity(command: string): SecurityResult
  protected abstract *runCommand(input: ShellInput, context: ToolUseContext): AsyncGenerator<ShellProgress>
}

// BashTool and PowerShellTool extend this
```

---

## Type Guards and Assertions

```typescript
// Type guards for message types
export function isUserMessage(msg: Message): msg is UserMessage {
  return msg.type === 'user'
}

export function isAssistantMessage(msg: Message): msg is AssistantMessage {
  return msg.type === 'assistant'
}

export function isToolResultMessage(msg: Message): msg is ToolResultMessage {
  return msg.type === 'tool_result'
}

// Usage
function processMessage(msg: Message) {
  if (isUserMessage(msg)) {
    // TypeScript knows msg is UserMessage
    console.log(msg.attachments)
  }
}
```

---

## Generic Abstractions

### Result Type

```typescript
// Common pattern for operations that can fail
export type Result<T, E = Error> =
  | { ok: true; value: T }
  | { ok: false; error: E }

export async function safeExecute<T>(
  fn: () => Promise<T>
): Promise<Result<T>> {
  try {
    const value = await fn()
    return { ok: true, value }
  } catch (error) {
    return { ok: false, error }
  }
}

// Usage
const result = await safeExecute(() => readFile('foo.txt'))
if (result.ok) {
  console.log(result.value)
} else {
  console.error(result.error)
}
```

### Option Type

```typescript
// For values that might not exist
export type Option<T> = T | null

export function findTool(name: string): Option<Tool> {
  return tools.find(t => t.name === name) ?? null
}

// Usage
const tool = findTool('BashTool')
if (tool !== null) {
  // TypeScript knows tool is Tool, not Option<Tool>
  await tool.execute(input, context)
}
```

---

## Summary

The key abstractions are:

| Abstraction | Purpose | Key Interface |
|-------------|---------|--------------|
| Tool | Execute operations | `Tool.execute()` |
| Message | Represent conversation | `Message` union type |
| Context | Provide environment | `ToolUseContext` |
| State | Manage application state | `Store<T>` |
| Event | Stream data | `StreamEvent` union |
| Task | Background operations | `Task` interface |
| MCP | External integrations | `MCPServerConnection` |
| Permission | Control access | `PermissionState` |
| Hook | Extend behavior | `Hook.execute()` |

These abstractions define the contracts between components, enabling loose coupling and testability.
