# Important Functions: Key Algorithms and Logic

This document highlights the most important functions in Claude Code, explaining what they do and how they work.

## Core Query Functions

### 1. query() — Main Query Orchestrator

```typescript
// query.ts
export async function* query(
  params: QueryParams
): AsyncGenerator<QueryEvent> {
  // This is the heart of Claude Code's AI interaction
  
  // 1. Build conversation messages from state
  const messages = await buildConversationMessages(params)
  
  // 2. Add system context (git status, project info)
  const withContext = await buildQueryContext(messages)
  
  // 3. Add memory files (.claude/memory.md)
  const withMemory = await addMemoryFiles(withContext)
  
  // 4. Get available tools
  const tools = await getTools(params.appState)
  
  // 5. Create query engine
  const engine = new QueryEngine({
    messages: withMemory,
    model: params.model,
    tools,
  })
  
  // 6. Stream response, handling tool calls
  for await (const event of engine.stream()) {
    if (event.type === 'tool_call') {
      // Execute tool and add result to messages
      yield* handleToolCall(event, params)
    } else {
      yield event
    }
  }
}
```

**Key Insight**: This is an async generator that yields events as they happen, enabling real-time streaming to the UI.

---

### 2. buildQueryContext() — Context Building

```typescript
// context.ts
export async function buildQueryContext(
  messages: Message[]
): Promise<QueryContext> {
  // Run all context gatherers in parallel for speed
  const [gitStatus, branch, userName, projectType] = await Promise.all([
    getGitStatus(),           // git status --short
    getBranch(),              // git branch
    getGitUserName(),         // git config user.name
    detectProjectType(),      // Check package.json, etc.
  ])
  
  // Build context string
  const context = [
    '# Git Status',
    gitStatus ?? 'Not a git repository',
    '',
    '# Current Branch',
    branch,
    '',
    '# Git User',
    userName ?? 'Unknown',
    '',
    '# Project Type',
    projectType,
  ].join('\n')
  
  return { messages, systemPrompt: context }
}
```

**Key Insight**: Using `Promise.all()` means all context gathering happens in parallel, not sequentially.

---

### 3. streamMessage() — API Streaming

```typescript
// services/api/claude.ts
async function streamMessage(
  params: MessageParams
): Promise<AsyncIterable<StreamEvent>> {
  // 1. Normalize messages for API
  const normalized = normalizeMessagesForAPI(params.messages)
  
  // 2. Convert tools to API schema
  const apiTools = params.tools?.map(toolToAPISchema)
  
  // 3. Create streaming request
  const stream = await client.messages.stream({
    model: params.model,
    max_tokens: 8192,
    messages: normalized,
    tools: apiTools,
  })
  
  return stream
}
```

---

## Tool System Functions

### 4. runTools() — Tool Orchestration

```typescript
// services/tools/toolOrchestration.ts
export async function* runTools(
  toolUseMessages: ToolUseBlock[],
  assistantMessages: AssistantMessage[],
  canUseTool: CanUseToolFn,
  toolUseContext: ToolUseContext,
): AsyncGenerator<MessageUpdate> {
  // Partition tools by concurrency safety
  const { safeTools, unsafeTools } = partitionToolCalls(toolUseMessages)
  
  // Read-only tools can run in parallel
  if (safeTools.length > 0) {
    const results = await Promise.all(
      safeTools.map(tool => runToolUse(tool, toolUseContext))
    )
    for (const result of results) {
      yield { message: result }
    }
  }
  
  // State-changing tools must run serially
  for (const tool of unsafeTools) {
    for await (const update of runToolUse(tool, toolUseContext)) {
      yield update
    }
  }
}

function partitionToolCalls(
  toolUses: ToolUseBlock[]
): { safeTools: ToolUseBlock[], unsafeTools: ToolUseBlock[] } {
  return {
    safeTools: toolUses.filter(t => READ_ONLY_TOOLS.has(t.name)),
    unsafeTools: toolUses.filter(t => !READ_ONLY_TOOLS.has(t.name)),
  }
}
```

**Key Insight**: This is where Claude Code gets its parallelism — read-only tools run together, but stateful tools run one at a time to prevent race conditions.

---

### 5. executeToolUse() — Individual Tool Execution

```typescript
// services/tools/toolExecution.ts
export async function* runToolUse(
  toolUse: ToolUseBlock,
  context: ToolUseContext,
): AsyncGenerator<MessageUpdate> {
  // 1. Find tool implementation
  const tool = findToolByName(toolUse.name)
  if (!tool) {
    yield createErrorMessage(`Tool not found: ${toolUse.name}`)
    return
  }
  
  // 2. Check permission
  const canUse = await context.canUseTool(tool.name)
  if (!canUse) {
    yield* requestPermission(tool, context)
    return
  }
  
  // 3. Execute with progress streaming
  const result = yield* executeWithProgress(tool, toolUse.input, context)
  
  // 4. Create result message
  yield {
    message: createToolResultMessage(toolUse.id, result),
  }
}
```

---

### 6. checkCommandSecurity() — Bash Security

```typescript
// tools/BashTool/bashSecurity.ts
export function checkCommandSecurity(
  command: string
): SecurityCheckResult {
  // Check for fork bombs
  if (command.includes(':(){:|:&};:')) {
    return { safe: false, reason: 'Fork bomb detected' }
  }
  
  // Check for rm -rf / or similar
  if (/rm\s+-rf\s+(\/|--no-preserve-root)/.test(command)) {
    return { safe: false, reason: 'Dangerous rm command' }
  }
  
  // Check for pipe to shell (curl | sh, etc.)
  if (/curl\s+.*\|\s*sh/.test(command)) {
    return { 
      safe: false, 
      reason: 'Pipe to shell detected - security risk' 
    }
  }
  
  // Check for dd to disk
  if (/dd\s+if=.*of=\/dev\//.test(command)) {
    return { 
      safe: false, 
      reason: 'Direct disk write detected' 
    }
  }
  
  return { safe: true }
}
```

**Key Insight**: Security is enforced at the tool level, preventing dangerous operations even if the AI model tries to execute them.

---

## Context Management Functions

### 7. compactMessages() — Context Compaction

```typescript
// services/compact/compact.ts
export async function compactMessages(
  messages: Message[],
  maxTokens: number
): Promise<Message[]> {
  // 1. Find the compact boundary
  // We want to summarize older messages that are still relevant
  const boundary = findCompactBoundary(messages)
  
  // 2. Extract messages to summarize
  const olderMessages = messages.slice(0, boundary)
  const recentMessages = messages.slice(boundary)
  
  // 3. Ask Claude to summarize
  const summary = await summarizeMessages(olderMessages)
  
  // 4. Create compact boundary message
  const boundaryMsg = createCompactBoundaryMessage(summary)
  
  // 5. Return compacted conversation
  return [boundaryMsg, ...recentMessages]
}

function findCompactBoundary(messages: Message[]): number {
  // Never compact recent messages or tool results
  for (let i = messages.length - 1; i >= 0; i--) {
    const msg = messages[i]
    
    // Don't compact recent user/assistant pairs
    if (isRecent(msg)) continue
    
    // Don't compact tool results
    if (msg.type === 'tool_result') continue
    
    // This is a safe boundary
    return i
  }
  
  // Fallback: compact older half
  return Math.floor(messages.length / 2)
}
```

**Key Insight**: Compaction preserves recent context while summarizing older content, maintaining conversation continuity.

---

### 8. normalizeMessagesForAPI() — Message Normalization

```typescript
// utils/messages.ts
export function normalizeMessagesForAPI(
  messages: Message[]
): NormalizedMessage[] {
  return messages.map(msg => {
    switch (msg.type) {
      case 'user':
        return {
          role: 'user',
          content: normalizeContent(msg.content),
        }
      case 'assistant':
        return {
          role: 'assistant',
          content: normalizeAssistantContent(msg),
        }
      case 'tool_result':
        return {
          role: 'user',
          content: [{
            type: 'tool_result',
            tool_use_id: msg.toolUseId,
            content: msg.content,
          }],
        }
      default:
        return null
    }
  }).filter(Boolean)
}
```

**Key Insight**: Claude Code uses a richer internal message format than the API expects, so normalization is required.

---

## State Management Functions

### 9. createStore() — Simple Reactive Store

```typescript
// state/store.ts
export function createStore<T>(
  initialState: T,
  onChange?: OnChange<T>,
): Store<T> {
  let state = initialState
  const listeners = new Set<Listener>()

  return {
    getState: () => state,
    
    setState: (updater) => {
      const prev = state
      const next = updater(prev)
      
      // Skip if no change (optimization!)
      if (Object.is(next, prev)) return
      
      state = next
      onChange?.({ newState: next, oldState: prev })
      
      // Notify all listeners
      for (const listener of listeners) {
        listener()
      }
    },
    
    subscribe: (listener) => {
      listeners.add(listener)
      return () => listeners.delete(listener)
    },
  }
}
```

**Key Insight**: Only ~35 lines but provides all the reactivity needed. Using `Object.is` to skip unnecessary updates is a key optimization.

---

### 10. applyEdit() — File Editing

```typescript
// tools/FileEditTool/utils.ts
export function applyEdits(
  original: string,
  edits: Edit[]
): string {
  // Sort edits in reverse order to apply from end to start
  // This prevents line number shifting issues
  const sortedEdits = [...edits].sort((a, b) => 
    (b.startLine ?? 0) - (a.startLine ?? 0)
  )
  
  let result = original
  for (const edit of sortedEdits) {
    switch (edit.type) {
      case 'replace':
        result = replaceRange(result, edit.start, edit.end, edit.newText)
        break
      case 'insert':
        result = insertAtLine(result, edit.line, edit.newText)
        break
      case 'delete':
        result = deleteLines(result, edit.startLine, edit.endLine)
        break
    }
  }
  
  return result
}
```

**Key Insight**: Edits are applied in reverse order to prevent line number shifting when multiple edits are made.

---

## API Functions

### 11. withRetry() — Retry Logic

```typescript
// services/api/withRetry.ts
export async function withRetry<T>(
  fn: () => Promise<T>,
  options: RetryOptions = {}
): Promise<T> {
  const { maxRetries = 3, baseDelay = 1000 } = options
  
  for (let attempt = 0; attempt <= maxRetries; attempt++) {
    try {
      return await fn()
    } catch (error) {
      if (!isRetryableError(error)) {
        throw error
      }
      
      if (attempt === maxRetries) {
        throw error
      }
      
      // Exponential backoff
      const delay = baseDelay * Math.pow(2, attempt)
      await sleep(delay)
    }
  }
  
  throw new Error('Unreachable')
}

function isRetryableError(error: APIError): boolean {
  // Retry on rate limits and server errors
  return error.statusCode === 429 ||
         error.statusCode >= 500
}
```

**Key Insight**: Exponential backoff prevents hammering the API during outages while eventually succeeding.

---

### 12. getGitStatus() — Git Information

```typescript
// context.ts
export const getGitStatus = memoize(async (): Promise<string | null> => {
  // Check if git repo
  if (!await getIsGit()) {
    return null
  }
  
  // Run git commands in parallel
  const [branch, status, recentLog, userName] = await Promise.all([
    execFile('git', ['branch', '--show-current']),
    execFile('git', ['status', '--short']),
    execFile('git', ['log', '--oneline', '-n', '5']),
    execFile('git', ['config', 'user.name']),
  ])
  
  return formatGitStatus(branch, status, recentLog, userName)
})
```

**Key Insight**: Using `memoize()` means git status is only computed once, even if called multiple times.

---

## Algorithm Patterns

### Token Estimation

```typescript
// utils/tokens.ts
export function estimateTokenCount(text: string): number {
  // Rough estimation: ~4 characters per token for English
  // This is a simplification; actual tokenization is more complex
  return Math.ceil(text.length / 4)
}

// More accurate estimation
export function accurateTokenCount(text: string): number {
  // Count words
  const words = text.split(/\s+/).length
  
  // Count special characters
  const specialChars = (text.match(/[^\w\s]/g) || []).length
  
  // Estimate tokens
  // Average English word is ~1.3 tokens
  // Special characters are ~2 tokens each
  return Math.ceil(words * 1.3 + specialChars * 2)
}
```

---

### Backoff Strategy

```typescript
// bridge/bridgeMain.ts
const DEFAULT_BACKOFF: BackoffConfig = {
  connInitialMs: 2_000,      // 2 seconds
  connCapMs: 120_000,        // 2 minutes max
  connGiveUpMs: 600_000,      // 10 minutes give up
  
  generalInitialMs: 500,      // 500ms for general ops
  generalCapMs: 30_000,       // 30 seconds max
}

function calculateBackoff(attempt: number, config: BackoffConfig): number {
  const delay = config.initialMs * Math.pow(2, attempt)
  return Math.min(delay, config.capMs)
}
```

---

### Task ID Generation

```typescript
// Task.ts
const TASK_ID_ALPHABET = '0123456789abcdefghijklmnopqrstuvwxyz'

export function generateTaskId(type: TaskType): string {
  const prefix = TASK_ID_PREFIXES[type] ?? 'x'
  const bytes = randomBytes(8)
  
  const id = Array.from(bytes)
    .map(b => TASK_ID_ALPHABET[b % 36])
    .join('')
  
  return `${prefix}${id}`
}
```

**Key Insight**: Task IDs use a prefix to identify the type, followed by random characters. The alphabet is case-insensitive to avoid issues in filenames/terminals.

---

## Summary

These functions demonstrate key algorithmic patterns:

1. **Async generators** — Streaming throughout
2. **Parallel execution** — `Promise.all()` for concurrent operations
3. **Memoization** — Cache expensive computations
4. **Retry with backoff** — Handle transient failures
5. **Partitioning** — Group by concurrency safety
6. **Normalization** — Convert between formats
7. **Boundary finding** — Safe points for compaction
8. **Reverse application** — Prevent index shifting
