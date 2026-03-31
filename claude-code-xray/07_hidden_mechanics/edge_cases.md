# Edge Cases: Handling the Unexpected

This document explains how Claude Code handles various edge cases, error scenarios, and unusual situations.

## 1. Context Window Overflow

### Scenario: Conversation exceeds model context limit.

**Handling**:

```typescript
// services/compact/compact.ts
export async function compactMessages(
  messages: Message[],
  maxTokens: number
): Promise<Message[]> {
  // Check if we need compaction
  const currentTokens = estimateTokens(messages)
  
  if (currentTokens <= maxTokens * 0.85) {
    return messages  // Plenty of room
  }
  
  // Find boundary for summarization
  const boundary = findCompactBoundary(messages)
  
  // Extract messages to summarize
  const olderMessages = messages.slice(0, boundary)
  const recentMessages = messages.slice(boundary)
  
  // Ask Claude to summarize
  const summary = await summarizeMessages(olderMessages)
  
  // Create boundary message
  const boundaryMsg = createCompactBoundaryMessage({
    summary,
    originalCount: olderMessages.length,
    summaryTokens: estimateTokens(summary),
  })
  
  return [boundaryMsg, ...recentMessages]
}

// Boundary finding logic
function findCompactBoundary(messages: Message[]): number {
  // Never compact:
  // - Most recent messages (active task context)
  // - Tool results (needed for tool use)
  // - The last N user/assistant pairs
  
  for (let i = messages.length - 1; i >= 0; i--) {
    const msg = messages[i]
    
    // Skip recent content
    if (isRecent(msg)) continue
    
    // Skip tool results
    if (msg.type === 'tool_result') continue
    
    // Found safe boundary
    return i
  }
  
  // Fallback
  return Math.floor(messages.length / 2)
}
```

**Edge Cases**:
- Single very long message → Split or truncate
- All messages recent → Can't compact, warn user
- Already compacted before → Don't compact again immediately

---

## 2. Tool Execution Timeout

### Scenario: A tool takes too long to execute.

**Handling**:

```typescript
// tools/BashTool/BashTool.ts
export async function executeCommand(
  command: string,
  context: ToolUseContext,
  timeout: number = 60000
): Promise<CommandResult> {
  const controller = new AbortController()
  
  // Set timeout
  const timeoutId = setTimeout(() => {
    controller.abort()
  }, timeout)
  
  try {
    const result = await spawn(command, {
      signal: controller.signal,
      cwd: context.cwd,
    })
    
    clearTimeout(timeoutId)
    return result
    
  } catch (error) {
    clearTimeout(timeoutId)
    
    if (error.name === 'AbortError') {
      return {
        success: false,
        error: `Command timed out after ${timeout}ms`,
        timedOut: true,
      }
    }
    
    throw error
  }
}
```

**User Options**:
- Default timeout: 60 seconds
- Can be overridden per command
- Infinite timeout for known long-running tasks
- Ctrl+C to abort manually

---

## 3. Permission Denial Cascade

### Scenario: User denies multiple permissions in a row.

**Handling**:

```typescript
// utils/permissions/denialTracking.ts
export interface DenialTrackingState {
  denials: DenialRecord[]
  consecutiveDenials: number
  escalationTriggered: boolean
}

export function trackDenial(
  state: DenialTrackingState,
  denial: DenialRecord
): DenialTrackingState {
  const newDenials = [...state.denials, denial]
  const consecutive = denial.timestamp - state.denials.at(-1)?.timestamp < 30000
    ? state.consecutiveDenials + 1
    : 1
  
  // After 3 consecutive denials, suggest alternatives
  const shouldEscalate = consecutive >= 3
  
  if (shouldEscalate && !state.escalationTriggered) {
    return {
      denials: newDenials,
      consecutiveDenials: consecutive,
      escalationTriggered: true,
      suggestion: suggestAlternative(denial.toolName),
    }
  }
  
  return {
    ...state,
    denials: newDenials,
    consecutiveDenials: consecutive,
  }
}
```

**What User Sees**:
```
Tool: BashTool (git push)
Permission: DENIED

Tool: BashTool (git push origin main)
Permission: DENIED

⚠️ You've denied 3 tool executions in a row.
Suggestions:
• Type /permissions to review your settings
• Use --no-edit flag to skip confirmation
```

---

## 4. Network Disconnection

### Scenario: Network fails during API call or remote session.

**Handling**:

```typescript
// bridge/bridgeMain.ts
const BACKOFF_CONFIG = {
  initialMs: 2_000,
  capMs: 120_000,
  giveUpMs: 600_000,
}

async function connectWithBackoff(
  config: ConnectionConfig
): Promise<Connection> {
  let attempt = 0
  
  while (true) {
    try {
      return await connect(config)
    } catch (error) {
      if (!isRetryableError(error)) {
        throw error
      }
      
      const backoff = Math.min(
        BACKOFF_CONFIG.initialMs * Math.pow(2, attempt),
        BACKOFF_CONFIG.capMs
      )
      
      if (elapsedTime > BACKOFF_CONFIG.giveUpMs) {
        throw new ConnectionFailedError('Maximum retry time exceeded')
      }
      
      console.log(`Retrying in ${backoff}ms...`)
      await sleep(backoff)
      attempt++
    }
  }
}
```

**User Experience**:
```
Connecting...
Connection lost. Retrying in 2s...
Connection lost. Retrying in 4s...
Connection lost. Retrying in 8s...
...
Connection lost. Retrying in 120s...
Maximum retry time exceeded.
Use /reconnect to try again or /local to switch to local mode.
```

---

## 5. File Conflicts

### Scenario: File is modified while Claude Code is editing it.

**Handling**:

```typescript
// tools/FileEditTool/FileEditTool.ts
export async function editFile(
  path: string,
  expectedContent: string,
  edits: Edit[]
): Promise<EditResult> {
  // Read current content
  const currentContent = await readFile(path)
  
  // Check for conflict
  if (currentContent !== expectedContent) {
    // File has changed
    const conflict = {
      type: 'file_conflict',
      expected: expectedContent,
      actual: currentContent,
      diff: computeDiff(expectedContent, currentContent),
    }
    
    return {
      success: false,
      error: 'File was modified since read',
      conflict,
      options: [
        'Retry with new content',
        'View diff',
        'Abort',
      ],
    }
  }
  
  // Apply edits
  const newContent = applyEdits(currentContent, edits)
  await writeFile(path, newContent)
  
  return { success: true, newContent }
}
```

**User Sees**:
```
⚠️ File conflict detected

File: src/utils/helper.ts
Modified by: Another process

Your changes:
  - Removed unused import
  + Added error handling

Current file:
  - Some other changes here

Options:
  1. Retry (re-read and apply your changes)
  2. View diff
  3. Abort
```

---

## 6. MCP Server Disconnection

### Scenario: MCP server crashes or disconnects.

**Handling**:

```typescript
// services/mcp/client.ts
export class MCPClient {
  private reconnectAttempts = 0
  private maxReconnectAttempts = 3
  
  async handleDisconnect(error: Error): Promise<void> {
    // Mark server as disconnected
    this.server.status = 'disconnected'
    
    // Try to reconnect
    while (this.reconnectAttempts < this.maxReconnectAttempts) {
      try {
        await this.connect()
        this.reconnectAttempts = 0
        return
      } catch (reconnectError) {
        this.reconnectAttempts++
        await sleep(1000 * this.reconnectAttempts)
      }
    }
    
    // Give up
    this.server.status = 'failed'
    
    // Notify user
    this.notifyError(
      `MCP server '${this.server.name}' disconnected. ` +
      `Tools from this server are unavailable.`
    )
  }
}
```

**User Sees**:
```
⚠️ MCP Server Disconnected

Server: github-mcp
Tools affected: /search, /issues, /pr

The server stopped responding. You can:
• /mcp restart github-mcp
• /mcp list to see other servers
```

---

## 7. Rate Limit Handling

### Scenario: API rate limit is hit.

**Handling**:

```typescript
// services/api/rateLimitHandling.ts
export class RateLimitHandler {
  private limits: Map<string, RateLimit> = new Map()
  
  handleResponse(headers: Headers, model: string): void {
    const remaining = parseInt(headers.get('x-ratelimit-remaining') ?? '9999')
    const resetAt = parseInt(headers.get('x-ratelimit-reset') ?? '0')
    
    this.limits.set(model, {
      remaining,
      resetAt: resetAt * 1000,  // Convert to ms
    })
  }
  
  async waitIfNeeded(model: string): Promise<void> {
    const limit = this.limits.get(model)
    if (!limit) return
    
    const now = Date.now()
    if (limit.remaining <= 0 && now < limit.resetAt) {
      const waitTime = limit.resetAt - now
      console.log(`Rate limited. Waiting ${waitTime}ms...`)
      await sleep(waitTime)
    }
  }
}
```

**User Sees**:
```
Waiting for rate limit reset...
Rate limit will reset in 45 seconds.

💡 Tips to avoid rate limits:
• Use a more specific prompt
• Process files in batches
• Consider upgrading your plan
```

---

## 8. Circular Tool Dependencies

### Scenario: Tool A needs tool B's result, but tool B needs tool A's context.

**Handling**:

```typescript
// services/tools/toolExecution.ts
// Detect circular dependencies before execution
export function detectCircularDependencies(
  toolCalls: ToolCall[]
): CircularDependency | null {
  const graph = new Map<string, string[]>()
  
  // Build dependency graph
  for (const call of toolCalls) {
    const deps = inferDependencies(call)
    graph.set(call.id, deps)
  }
  
  // Detect cycles using DFS
  const visited = new Set<string>()
  const recursionStack = new Set<string>()
  
  for (const [id] of graph) {
    if (hasCycle(id)) {
      return extractCycle(id)
    }
  }
  
  return null
  
  function hasCycle(nodeId: string): boolean {
    if (recursionStack.has(nodeId)) {
      return true  // Cycle found
    }
    if (visited.has(nodeId)) {
      return false
    }
    
    visited.add(nodeId)
    recursionStack.add(nodeId)
    
    for (const dep of graph.get(nodeId) ?? []) {
      if (hasCycle(dep)) {
        return true
      }
    }
    
    recursionStack.delete(nodeId)
    return false
  }
}
```

**Resolution**:
- If circular dependency detected, execute in topological order
- Some tools might need re-ordering
- Worst case: ask user to restructure prompt

---

## 9. Empty Tool Results

### Scenario: A tool returns empty or null.

**Handling**:

```typescript
// services/tools/toolExecution.ts
export function formatToolResult(result: unknown): string {
  if (result === null || result === undefined) {
    return '(No output)'
  }
  
  if (typeof result === 'string') {
    return result || '(Empty string)'
  }
  
  if (Array.isArray(result)) {
    if (result.length === 0) {
      return '(No items)'
    }
    return result.map(formatToolResult).join('\n')
  }
  
  if (typeof result === 'object') {
    return JSON.stringify(result, null, 2)
  }
  
  return String(result)
}
```

**Why**: Empty results are still results. Claude needs to know something happened (even if nothing was output).

---

## 10. Malformed Tool Input

### Scenario: Claude sends invalid input to a tool.

**Handling**:

```typescript
// tools/FileEditTool/FileEditTool.ts
export function validateEditInput(input: unknown): ValidationResult {
  const schema = {
    type: 'object',
    required: ['path'],
    properties: {
      path: { type: 'string' },
      edits: {
        type: 'array',
        items: {
          type: 'object',
          required: ['type'],
          properties: {
            type: { enum: ['replace', 'insert', 'delete'] },
            // ... more schema
          },
        },
      },
    },
  }
  
  try {
    const validated = JSONSchema.validate(input, schema)
    return { valid: true, value: validated }
  } catch (error) {
    return {
      valid: false,
      error: formatValidationError(error),
    }
  }
}

// When invalid input received
export async function handleInvalidInput(
  tool: Tool,
  input: unknown
): Promise<ToolResult> {
  const validation = tool.validateInput(input)
  
  return {
    success: false,
    error: `Invalid input for ${tool.name}: ${validation.error}`,
    suggestion: `Expected format: ${JSON.stringify(tool.inputSchema)}`,
  }
}
```

---

## 11. Concurrent State Updates

### Scenario: Multiple updates to state at the same time.

**Handling**:

```typescript
// state/store.ts
export function createStore<T>(initialState: T): Store<T> {
  let state = state
  
  return {
    setState(updater) {
      const prev = state
      const next = updater(prev)
      
      // Compare by reference first (fast path)
      if (Object.is(next, prev)) {
        return  // No change, skip
      }
      
      // Deep equality check for complex objects
      if (deepEqual(next, prev)) {
        return  // No semantic change, skip
      }
      
      state = next
      notifyAllListeners()
    },
  }
}
```

**Why**: Two concurrent updates shouldn't cause race conditions. The synchronous `setState` prevents this.

---

## 12. Session Restore Failure

### Scenario: Previous session can't be restored.

**Handling**:

```typescript
// utils/sessionStorage.ts
export async function restoreSession(
  sessionId: string
): Promise<RestoreResult> {
  try {
    const snapshot = await loadSession(sessionId)
    
    // Validate snapshot
    if (!snapshot) {
      return { success: false, error: 'Session not found' }
    }
    
    if (!snapshot.messages?.length) {
      return { success: false, error: 'Session has no messages' }
    }
    
    // Check for corrupted data
    for (const msg of snapshot.messages) {
      if (!isValidMessage(msg)) {
        return {
          success: false,
          error: 'Session contains corrupted data',
          partialRestore: true,
          validMessages: snapshot.messages.filter(isValidMessage),
        }
      }
    }
    
    return { success: true, snapshot }
    
  } catch (error) {
    return {
      success: false,
      error: `Failed to restore: ${error.message}`,
    }
  }
}
```

**User Options**:
```
⚠️ Session Restore Failed

Session: abc123
Error: Corrupted message data

Partial restore available: 47/50 messages

Options:
  1. Restore partial session (47 messages)
  2. Start fresh session
  3. View corrupted messages
```

---

## Summary

Claude Code handles edge cases through:

| Edge Case | Strategy |
|-----------|----------|
| Context overflow | Compaction with summarization |
| Tool timeout | Abort controller + user options |
| Permission denial cascade | Tracking + escalation |
| Network failure | Backoff retry |
| File conflicts | Detection + resolution options |
| MCP disconnect | Auto-reconnect + notification |
| Rate limits | Wait + user education |
| Circular deps | Detection + topological sort |
| Empty results | Format with context |
| Invalid input | Schema validation + helpful errors |
| Concurrent updates | Synchronous setState |
| Session restore | Validation + partial restore |
