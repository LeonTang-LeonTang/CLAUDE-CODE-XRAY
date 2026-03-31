# Non-Obvious Behaviors: The Hidden Gems

This document reveals the surprising, clever, and non-obvious behaviors in Claude Code that you might not discover by just using it.

## 1. Prefetch Optimization at Startup

**What You See**: Claude Code starts up quickly.

**What's Actually Happening**:

```typescript
// main.tsx - BEFORE all imports!
// These run while modules are still loading

profileCheckpoint('main_tsx_entry')

// Fire off MDM reads (macOS device management)
// This starts subprocesses, runs in parallel with imports
startMdmRawRead()

// Start keychain prefetch
// Reads OAuth tokens while modules are loading
// Without this: 65ms sequential reads EVERY startup
startKeychainPrefetch()

// Heavy imports happen here (~135ms)
// Meanwhile, keychain and MDM are already done
import { /* 50+ modules */ } from './...'
```

**💡 Insight**: The very first lines of main.tsx (before imports!) are doing I/O that runs in parallel with module loading. This is a ~200ms optimization.

---

## 2. Dead Code Elimination via Feature Flags

**What You See**: Different Claude Code builds have different features.

**What's Actually Happening**:

```typescript
import { feature } from 'bun:bundle'

// In builds WITHOUT KAIROS, this entire import is removed
const SleepTool = feature('KAIROS')
  ? require('./tools/SleepTool/SleepTool.js').SleepTool
  : null

// In builds WITHOUT COORDINATOR_MODE, this module doesn't exist
const coordinator = feature('COORDINATOR_MODE')
  ? require('./coordinator/coordinatorMode.js')
  : null
```

**Why**: `feature()` is a compile-time function. In builds without the feature, Bun removes the entire conditional. No runtime check overhead.

---

## 3. Async Generators Throughout

**What You See**: Streaming responses and real-time tool progress.

**What's Actually Happening**: Claude Code uses async generators at every layer:

```typescript
// query.ts - Main query is an async generator
export async function* query(params): AsyncGenerator<QueryEvent> {
  for await (const event of engine.stream()) {
    yield event  // Yields as events arrive
  }
}

// Query engine is also an async generator
async *stream(): AsyncGenerator<StreamEvent> {
  for await (const rawEvent of apiStream) {
    yield processEvent(rawEvent)
  }
}

// Tools use async generators for progress
async *execute(input, context): AsyncGenerator<Progress> {
  yield { type: 'starting' }
  await setup()
  yield { type: 'running', progress: 50 }
  await complete()
  yield { type: 'done' }
}
```

**💡 Insight**: The entire flow is streaming. Nothing is buffered. The UI receives updates the moment they happen.

---

## 4. Concurrency Partitioning

**What You See**: Multiple read-only tools run at the same time.

**What's Actually Happening**:

```typescript
// toolOrchestration.ts
export async function* runTools(toolCalls) {
  // Partition by concurrency safety
  const { safeTools, unsafeTools } = partitionBySafety(toolCalls)
  
  // Read-only tools can run concurrently
  if (safeTools.length > 0) {
    yield* runToolsConcurrently(safeTools)
  }
  
  // Stateful tools MUST run serially
  if (unsafeTools.length > 0) {
    yield* runToolsSerially(unsafeTools)
  }
}

// Claude asks for file reads + grep + glob
// These all run in PARALLEL because they're read-only
const toolCalls = [
  { name: 'FileReadTool', ... },
  { name: 'GrepTool', ... },
  { name: 'GlobTool', ... },
]
```

**Why**: Multiple file reads can happen simultaneously without affecting each other. But two file writes could corrupt data if interleaved.

---

## 5. Reverse Edit Application

**What You See**: Multiple file edits work correctly.

**What's Actually Happening**:

```typescript
// tools/FileEditTool/utils.ts
export function applyEdits(original, edits) {
  // Sort edits in REVERSE order!
  const sorted = [...edits].sort((a, b) => 
    (b.startLine ?? 0) - (a.startLine ?? 0)
  )
  
  let result = original
  for (const edit of sorted) {
    result = applySingleEdit(result, edit)
  }
  return result
}

// Example: Edit lines 5 and 10
// Wrong order: Edit line 5 first → line 10 shifts to 11
// Right order: Edit line 10 first → line 5 stays at 5
```

**💡 Insight**: By applying from bottom to top, line numbers don't shift during the loop.

---

## 6. Memoized Git Status

**What You See**: Git status is fast even when queried multiple times.

**What's Actually Happening**:

```typescript
// context.ts
import memoize from 'lodash-es/memoize.js'

// getGitStatus is called multiple times per query
// But git commands only run ONCE
export const getGitStatus = memoize(async (): Promise<string | null> => {
  // Expensive git commands here
  const [branch, status] = await Promise.all([
    execFile('git', ['branch', '--show-current']),
    execFile('git', ['status', '--short']),
  ])
  return formatGitStatus(branch, status)
})
```

**Why**: Claude Code might call getGitStatus() 3-4 times per query. Memoization means git only runs once per session.

---

## 7. Context Compaction with Boundary Messages

**What You See**: Long conversations continue working even after many exchanges.

**What's Actually Happening**:

```typescript
// services/compact/compact.ts
export async function compactMessages(messages) {
  // Find boundary (oldest message safe to summarize)
  const boundary = findCompactBoundary(messages)
  
  // Summarize older messages
  const olderMessages = messages.slice(0, boundary)
  const summary = await summarizeWithClaude(olderMessages)
  
  // Insert boundary message
  const boundaryMsg = {
    type: 'system_compact_boundary',
    summary: summary,
    originalMessageCount: boundary,
  }
  
  // Remove originals, keep boundary
  return [boundaryMsg, ...messages.slice(boundary)]
}
```

**💡 Insight**: The boundary message contains the summary. Future queries see the summary instead of 100 individual messages.

---

## 8. Permission State Machine

**What You See**: Permission prompts appear at the right time.

**What's Actually Happening**:

```typescript
// Permission is a state machine
type PermissionState =
  | { status: 'ready' }
  | { status: 'requesting'; toolName: string }
  | { status: 'approved'; toolName: string; remember: boolean }
  | { status: 'denied'; toolName: string }

// Transitions are enforced
function transition(state, event): PermissionState {
  switch (state.status) {
    case 'ready':
      if (event.type === 'tool_called') {
        return { status: 'requesting', toolName: event.tool }
      }
    case 'requesting':
      if (event.type === 'approve') {
        return { 
          status: 'approved', 
          toolName: state.toolName,
          remember: event.remember 
        }
      }
      if (event.type === 'deny') {
        return { status: 'denied', toolName: state.toolName }
      }
    // ...
  }
}
```

**Why**: The state machine prevents impossible transitions (can't go from 'ready' to 'denied' without 'requesting').

---

## 9. React Without DOM (Ink)

**What You See**: A React-like component tree renders to terminal.

**What's Actually Happening**:

```typescript
// ink/reconciler.ts
// Instead of React DOM, there's a custom reconciler

// Components still use JSX
const App = () => (
  <Box flexDirection="column">
    <Text>Hello</Text>
  </Box>
)

// But the reconciler outputs ANSI, not HTML
function reconcile(element) {
  const node = Yoga.Node.create()
  
  // Calculate layout with Yoga
  node.calculateLayout()
  
  // Output ANSI escape codes
  process.stdout.write(
    `\x1b[${node.top};${node.left}H${node.text}`
  )
}
```

**💡 Insight**: React components + Yoga layout engine + ANSI output = the terminal UI.

---

## 10. Token Estimation vs Tokenization

**What You See**: Token counts in usage stats.

**What's Actually Happening**:

```typescript
// utils/tokens.ts
// Real tokenization uses the API's tiktoken
import tiktoken from 'tiktoken'

// But for quick estimates:
export function estimateTokens(text: string): number {
  return Math.ceil(text.length / 4)  // Rough: 4 chars/token
}

// For accurate counts (used before API calls):
export function countTokens(text: string): number {
  const encoding = tiktoken.for_model('claude-3')
  return encoding.encode(text).length
}
```

**Why**: Quick estimates are fast but approximate. Accurate counts are slower but correct (used for context management).

---

## 11. Profile Checkpoints Throughout

**What You See**: Performance profiling in debug mode.

**What's Actually Happening**:

```typescript
// utils/startupProfiler.ts
export function profileCheckpoint(name: string): void {
  const now = performance.now()
  checkpoints.push({ name, startTime: now })
}

export function profileReport(): void {
  // Outputs timing breakdown
  // main_tsx_entry: 0.00ms
  // modules_loaded: 134.52ms
  // config_loaded: 10.21ms
  // auth_initialized: 15.33ms
}

// Used throughout startup
profileCheckpoint('main_tsx_entry')
startMdmRawRead()
profileCheckpoint('mdm_started')
// ...
```

**💡 Insight**: Built-in profiling lets developers see exactly where startup time is spent.

---

## 12. MCP Dynamic Tool Discovery

**What You See**: MCP servers appear as tools automatically.

**What's Actually Happening**:

```typescript
// services/mcp/client.ts
export class MCPClient {
  async connect(config) {
    const transport = this.createTransport(config)
    const client = new Client({ transport })
    
    // Initialize connection
    await client.initialize()
    
    // Discover tools
    const { tools } = await client.listTools()
    
    // Wrap each as a Claude Code tool
    for (const toolDef of tools) {
      const wrapperTool = {
        name: `mcp_${config.name}_${toolDef.name}`,
        description: toolDef.description,
        execute: async (input, ctx) => {
          const result = await client.callTool(toolDef.name, input)
          return formatMCPResult(result)
        },
      }
      registerTool(wrapperTool)
    }
  }
}
```

**Why**: MCP tools are discovered at runtime and wrapped as Claude Code tools. No code changes needed.

---

## 13. Sleep-to-Wake for Background Tasks

**What You See**: Background tasks run while you continue chatting.

**What's Actually Happening**:

```typescript
// bridge/capacityWake.ts
// When CPU is idle, background tasks might sleep
// Claude Code can "wake" them when needed

export function createCapacityWake(config): CapacityWake {
  return {
    // Check if task is sleeping
    async isSleeping(taskId): Promise<boolean> {
      return tasks.get(taskId)?.status === 'sleeping'
    },
    
    // Wake a sleeping task
    async wake(taskId): Promise<void> {
      const task = tasks.get(taskId)
      if (task?.status === 'sleeping') {
        await task.resume()
        tasks.set(taskId, { ...task, status: 'running' })
      }
    },
  }
}
```

---

## 14. Session Fingerprinting

**What You See**: Can resume previous sessions.

**What's Actually Happening**:

```typescript
// utils/sessionStorage.ts
export interface SessionSnapshot {
  id: string
  messages: Message[]
  // Fingerprint for cache validation
  fingerprint: string
  timestamp: Date
}

export function computeFingerprint(messages): string {
  // Hash of conversation state
  const content = JSON.stringify({
    messages: messages.map(m => ({
      type: m.type,
      contentLength: m.content.length,
    })),
  })
  return sha256(content)
}
```

**Why**: The fingerprint lets Claude Code detect if session state has changed since it was saved.

---

## Summary

These non-obvious behaviors show the depth of engineering:

1. **Prefetch I/O** — Background work before heavy imports
2. **Feature flags** — Compile-time DCE
3. **Async generators** — Streaming throughout
4. **Concurrency partitioning** — Safe parallelism
5. **Reverse edit application** — Correct multi-edit
6. **Memoization** — Cache expensive calls
7. **Compaction boundaries** — Long conversation support
8. **Permission state machine** — Enforced transitions
9. **Ink reconciler** — React-to-ANSI
10. **Token estimation** — Fast vs accurate
11. **Profile checkpoints** — Built-in profiling
12. **Dynamic MCP wrapping** — Runtime discovery
13. **Capacity wake** — Efficient background tasks
14. **Session fingerprinting** — Change detection
