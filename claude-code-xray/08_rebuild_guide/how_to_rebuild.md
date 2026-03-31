# How to Rebuild: Building Claude Code-Style Systems

This guide walks you through rebuilding key components of a Claude Code-style agentic CLI system. Each section builds on the previous, creating a complete working system.

## Project Structure

```
my-claude/
├── src/
│   ├── index.ts              # Entry point
│   ├── cli/                  # CLI parsing
│   ├── query/                # Query orchestration
│   ├── tools/               # Tool implementations
│   ├── state/                # State management
│   ├── api/                  # API client
│   └── terminal/             # Terminal UI
├── package.json
└── tsconfig.json
```

## Step 1: Minimal Reactive Store

Start with the simplest possible state management.

```typescript
// state/store.ts
export type Listener = () => void

export interface Store<T> {
  getState(): T
  setState(updater: (prev: T) => T): void
  subscribe(listener: Listener): () => void
}

export function createStore<T>(
  initialState: T,
  onChange?: (args: { newState: T; oldState: T }) => void
): Store<T> {
  let state = initialState
  const listeners = new Set<Listener>()

  return {
    getState: () => state,
    
    setState: (updater) => {
      const prev = state
      const next = updater(prev)
      
      // Skip if no change
      if (Object.is(next, prev)) return
      
      state = next
      onChange?.({ newState: next, oldState: prev })
      listeners.forEach(l => l())
    },
    
    subscribe: (listener) => {
      listeners.add(listener)
      return () => listeners.delete(listener)
    },
  }
}
```

**Test it**:

```typescript
// test-store.ts
const store = createStore({ count: 0 })

store.subscribe(() => {
  console.log('Changed:', store.getState())
})

store.setState(s => ({ count: s.count + 1 }))
// Output: Changed: { count: 1 }

store.setState(s => ({ count: s.count }))
// No output (Object.is caught the no-op)
```

---

## Step 2: Tool Interface

Define the tool contract that all tools must implement.

```typescript
// tools/types.ts
export interface ToolResult {
  content: string
  isError?: boolean
}

export interface ToolProgress {
  type: string
  [key: string]: unknown
}

export interface ToolUseContext {
  cwd: string
  abortSignal: AbortSignal
}

export interface Tool {
  name: string
  description: string
  inputSchema: {
    type: 'object'
    properties?: Record<string, unknown>
    required?: string[]
  }
  
  execute(
    input: unknown,
    context: ToolUseContext
  ): Promise<ToolResult> | AsyncGenerator<ToolProgress, ToolResult>
}
```

**Example tool**:

```typescript
// tools/echo.ts
export const echoTool = {
  name: 'EchoTool',
  description: 'Echoes the input back',
  inputSchema: {
    type: 'object',
    properties: {
      message: { type: 'string' }
    },
    required: ['message']
  },
  
  async execute(input: { message: string }): Promise<ToolResult> {
    return {
      content: `Echo: ${input.message}`
    }
  }
}
```

---

## Step 3: Tool Registry

Create a registry that holds all available tools.

```typescript
// tools/registry.ts
import type { Tool } from './types'

const tools = new Map<string, Tool>()

export function registerTool(tool: Tool): void {
  tools.set(tool.name, tool)
}

export function getTool(name: string): Tool | undefined {
  return tools.get(name)
}

export function getAllTools(): Tool[] {
  return Array.from(tools.values())
}

// Auto-register built-in tools
registerTool(echoTool)
```

---

## Step 4: Tool Orchestration

Build the pipeline that executes tools in the right order.

```typescript
// tools/orchestration.ts
import type { Tool, ToolResult, ToolProgress, ToolUseContext } from './types'
import { getTool } from './registry'

export async function* executeTool(
  toolName: string,
  input: unknown,
  context: ToolUseContext
): AsyncGenerator<ToolProgress, ToolResult> {
  const tool = getTool(toolName)
  
  if (!tool) {
    return { content: `Tool not found: ${toolName}`, isError: true }
  }
  
  // Validate input
  const validation = validateInput(tool, input)
  if (!validation.valid) {
    return { 
      content: `Invalid input: ${validation.error}`, 
      isError: true 
    }
  }
  
  // Execute tool
  const result = await tool.execute(validation.value, context)
  
  return result
}

function validateInput(
  tool: Tool, 
  input: unknown
): { valid: boolean; value?: unknown; error?: string } {
  // Simple validation - just check required fields
  const required = tool.inputSchema.required ?? []
  
  for (const field of required) {
    if (!input || !(field in (input as object))) {
      return { valid: false, error: `Missing required field: ${field}` }
    }
  }
  
  return { valid: true, value: input }
}
```

---

## Step 5: Concurrency Partitioning

Implement smart concurrency for parallel tool execution.

```typescript
// tools/concurrency.ts
// Tools that only read, never modify state
const READ_ONLY_TOOLS = new Set(['EchoTool', 'GrepTool', 'GlobTool', 'ReadTool'])

export function partitionByConcurrencySafety(
  toolCalls: Array<{ name: string; input: unknown }>
): {
  parallel: Array<{ name: string; input: unknown }>
  serial: Array<{ name: string; input: unknown }>
} {
  return {
    parallel: toolCalls.filter(t => READ_ONLY_TOOLS.has(t.name)),
    serial: toolCalls.filter(t => !READ_ONLY_TOOLS.has(t.name)),
  }
}

export async function* runTools(
  toolCalls: Array<{ name: string; input: unknown }>,
  context: ToolUseContext
): AsyncGenerator<{ toolName: string; progress?: ToolProgress; result?: ToolResult }> {
  const { parallel, serial } = partitionByConcurrencySafety(toolCalls)
  
  // Run parallel tools concurrently
  const parallelPromises = parallel.map(async (call) => {
    const result = await executeTool(call.name, call.input, context)
    return { toolName: call.name, result }
  })
  
  for (const promise of parallelPromises) {
    yield await promise
  }
  
  // Run serial tools one at a time
  for (const call of serial) {
    const result = yield* executeTool(call.name, call.input, context)
    yield { toolName: call.name, result }
  }
}
```

---

## Step 6: Simple API Client

Build a minimal Anthropic API client.

```typescript
// api/client.ts
export interface Message {
  role: 'user' | 'assistant' | 'system'
  content: string
}

export interface ToolParam {
  name: string
  description?: string
  input_schema: object
}

export interface APIOptions {
  apiKey: string
  model?: string
  maxTokens?: number
  tools?: ToolParam[]
}

export async function createMessage(
  messages: Message[],
  options: APIOptions
): Promise<{
  content: string
  stopReason: string
}> {
  const response = await fetch('https://api.anthropic.com/v1/messages', {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'x-api-key': options.apiKey,
      'anthropic-version': '2023-06-01',
    },
    body: JSON.stringify({
      model: options.model ?? 'claude-sonnet-4-7',
      max_tokens: options.maxTokens ?? 1024,
      messages,
      tools: options.tools,
    }),
  })
  
  if (!response.ok) {
    throw new Error(`API error: ${response.status}`)
  }
  
  const data = await response.json()
  return {
    content: data.content[0].text,
    stopReason: data.stop_reason,
  }
}
```

---

## Step 7: Streaming API Client

Upgrade to streaming for real-time responses.

```typescript
// api/streaming.ts
export type StreamEvent =
  | { type: 'text'; text: string }
  | { type: 'tool_use'; toolName: string; toolInput: object }
  | { type: 'complete'; stopReason: string }

export async function* streamMessage(
  messages: Message[],
  options: APIOptions
): AsyncGenerator<StreamEvent> {
  const response = await fetch('https://api.anthropic.com/v1/messages', {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'x-api-key': options.apiKey,
      'anthropic-version': '2023-06-01',
    },
    body: JSON.stringify({
      model: options.model ?? 'claude-sonnet-4-7',
      max_tokens: options.maxTokens ?? 1024,
      messages,
      tools: options.tools,
      stream: true,
    }),
  })
  
  if (!response.ok) {
    throw new Error(`API error: ${response.status}`)
  }
  
  // Parse SSE stream
  const reader = response.body?.getReader()
  if (!reader) throw new Error('No response body')
  
  const decoder = new TextDecoder()
  let buffer = ''
  
  while (true) {
    const { done, value } = await reader.read()
    
    if (done) break
    
    buffer += decoder.decode(value, { stream: true })
    
    // Process complete lines
    const lines = buffer.split('\n')
    buffer = lines.pop() ?? ''
    
    for (const line of lines) {
      if (!line.startsWith('data: ')) continue
      
      const data = JSON.parse(line.slice(6))
      yield parseEvent(data)
    }
  }
}

function parseEvent(data: any): StreamEvent {
  switch (data.type) {
    case 'content_block_delta':
      if (data.delta.type === 'text_delta') {
        return { type: 'text', text: data.delta.text }
      }
    case 'message_delta':
      return { type: 'complete', stopReason: data.delta.stop_reason }
    default:
      return { type: 'text', text: '' }
  }
}
```

---

## Step 8: Query Orchestration

Combine API client with tool execution.

```typescript
// query/orchestrator.ts
import type { StreamEvent } from '../api/streaming'
import { streamMessage, type Message } from '../api/client'
import { runTools, type ToolUseContext } from '../tools/orchestration'
import { getAllTools, type Tool } from '../tools/registry'

export interface QueryOptions {
  messages: Message[]
  apiKey: string
  model?: string
  tools?: Tool[]
}

export async function* query(
  options: QueryOptions
): AsyncGenerator<{ type: string; data: unknown }> {
  const tools = options.tools ?? getAllTools()
  
  // Convert tools to API format
  const apiTools = tools.map(t => ({
    name: t.name,
    description: t.description,
    input_schema: t.inputSchema,
  }))
  
  // Initial API call
  const context: ToolUseContext = { cwd: process.cwd(), abortSignal: new AbortController().signal }
  
  for await (const event of streamMessage(options.messages, {
    apiKey: options.apiKey,
    model: options.model,
    tools: apiTools,
  })) {
    if (event.type === 'text') {
      yield { type: 'text', data: event.text }
    } else if (event.type === 'tool_use') {
      // Execute tool
      const result = await executeTool(event.toolName, event.toolInput, context)
      yield { type: 'tool_result', data: result }
      
      // Add result to messages and continue
      options.messages.push({
        role: 'user',
        content: `Tool result: ${JSON.stringify(result)}`,
      })
    } else if (event.type === 'complete') {
      yield { type: 'complete', data: event.stopReason }
    }
  }
}
```

---

## Step 9: Terminal UI

Build a simple terminal UI with streaming output.

```typescript
// terminal/ui.ts
import readline from 'readline'

export interface TerminalUI {
  print(text: string): void
  printLine(text: string): void
  readLine(prompt: string): Promise<string>
  clear(): void
}

export function createTerminalUI(): TerminalUI {
  const rl = readline.createInterface({
    input: process.stdin,
    output: process.stdout,
  })
  
  return {
    print(text: string) {
      process.stdout.write(text)
    },
    
    printLine(text: string) {
      console.log(text)
    },
    
    async readLine(prompt: string): Promise<string> {
      return new Promise(resolve => {
        rl.question(prompt, resolve)
      })
    },
    
    clear() {
      process.stdout.write('\x1b[2J\x1b[H')
    },
  }
}

export async function runREPL(
  ui: TerminalUI,
  queryFn: (input: string) => AsyncGenerator<{ type: string; data: unknown }>
) {
  ui.printLine('Welcome to MyClaude! Type /exit to quit.\n')
  
  while (true) {
    const input = await ui.readLine('> ')
    
    if (input === '/exit') break
    
    if (input.trim()) {
      for await (const event of queryFn(input)) {
        if (event.type === 'text') {
          ui.print(event.data as string)
        } else if (event.type === 'tool_result') {
          ui.printLine(`\n[Tool: ${JSON.stringify(event.data)}]\n`)
        } else if (event.type === 'complete') {
          ui.printLine('')
        }
      }
    }
  }
  
  ui.printLine('Goodbye!')
}
```

---

## Step 10: Entry Point

Wire everything together.

```typescript
// index.ts
import { createStore } from './state/store'
import { registerTool } from './tools/registry'
import { createTerminalUI, runREPL } from './terminal/ui'
import { query } from './query/orchestrator'
import { echoTool } from './tools/echo'

// Register tools
registerTool(echoTool)

// Create state
const state = createStore({
  messages: [],
  isProcessing: false,
})

// Create UI
const ui = createTerminalUI()

// Run the REPL
runREPL(ui, async function* (input) {
  state.setState(s => ({
    ...s,
    messages: [...s.messages, { role: 'user', content: input }],
    isProcessing: true,
  }))
  
  const apiKey = process.env.ANTHROPIC_API_KEY
  if (!apiKey) {
    yield { type: 'error', data: 'ANTHROPIC_API_KEY not set' }
    return
  }
  
  for await (const event of query({
    messages: state.getState().messages,
    apiKey,
  })) {
    yield event
    
    if (event.type === 'complete') {
      state.setState(s => ({ ...s, isProcessing: false }))
    }
  }
})
```

---

## Step 11: Permission System

Add permission prompts for dangerous operations.

```typescript
// state/permissions.ts
export type PermissionState =
  | { status: 'ready' }
  | { status: 'requesting'; toolName: string; reason: string }

export interface PermissionOptions {
  toolName: string
  reason: string
  dangerous?: boolean
}

export async function requestPermission(
  ui: TerminalUI,
  options: PermissionOptions
): Promise<boolean> {
  const dangerous = options.dangerous ?? false
  
  ui.printLine(`\n${dangerous ? '⚠️ ' : ''}Permission requested: ${options.toolName}`)
  ui.printLine(`Reason: ${options.reason}\n`)
  
  const response = await ui.readLine('Allow? [y/N] ')
  
  return ['y', 'yes'].includes(response.toLowerCase())
}
```

---

## Step 12: Context Building

Add system context to queries.

```typescript
// context/builder.ts
import { exec } from 'child_process'
import { promisify } from 'util'

const execAsync = promisify(exec)

export async function getSystemContext(): Promise<string> {
  const [gitStatus, gitBranch] = await Promise.all([
    getGitStatus(),
    getGitBranch(),
  ])
  
  return `
Current Directory: ${process.cwd()}
Git Branch: ${gitBranch || 'Not a git repo'}
Git Status:
${gitStatus || 'No changes'}
`.trim()
}

async function getGitStatus(): Promise<string | null> {
  try {
    const { stdout } = await execAsync('git status --short 2>/dev/null')
    return stdout || null
  } catch {
    return null
  }
}

async function getGitBranch(): Promise<string | null> {
  try {
    const { stdout } = await execAsync('git branch --show-current 2>/dev/null')
    return stdout.trim() || null
  } catch {
    return null
  }
}
```

---

## Further Enhancements

With the core in place, you can add:

1. **File Tools**: Read, write, edit files
2. **Bash Tool**: Execute shell commands
3. **MCP Integration**: Dynamic tool discovery
4. **Context Compaction**: Handle long conversations
5. **Session Persistence**: Save and restore sessions
6. **Multi-Agent**: Spawn sub-agents
7. **Ink UI**: React-based terminal rendering
8. **Remote Sessions**: Bridge to remote Claude Code

## Testing Your Implementation

```typescript
// test/query.test.ts
async function testQuery() {
  const store = createStore({ messages: [] })
  
  // Mock API call
  global.fetch = async (url, options) => ({
    ok: true,
    json: async () => ({
      content: [{ type: 'text', text: 'Hello!' }],
      stop_reason: 'end_turn',
    }),
  })
  
  // Test the flow
  const context = await getSystemContext()
  console.log('Context:', context)
  
  store.setState(s => ({
    ...s,
    messages: [{ role: 'user', content: 'Hi' }],
  }))
  
  console.log('Final state:', store.getState())
}

testQuery()
```

## Summary

You now have:

1. **Reactive Store** — Simple state management
2. **Tool Interface** — Standard tool contract
3. **Tool Registry** — Centralized tool management
4. **Tool Orchestration** — Pipeline for execution
5. **Concurrency Partitioning** — Safe parallelism
6. **API Client** — Anthropic API integration
7. **Streaming Client** — Real-time responses
8. **Query Orchestrator** — Combines everything
9. **Terminal UI** — User interaction
10. **Permission System** — Safety gates
11. **Context Builder** — Rich system prompts

This is the foundation of a Claude Code-style system!
