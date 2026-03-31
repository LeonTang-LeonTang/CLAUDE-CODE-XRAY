# Critical Files: The Heart of Claude Code

This document details the most important files in Claude Code's codebase — the files that contain the core logic, coordination, and essential systems.

## File Importance Hierarchy

```
Tier 1: Critical (Can't run without these)
├── main.tsx           - Entry point
├── query.ts           - Query orchestration  
├── QueryEngine.ts     - Streaming engine
├── Tool.ts            - Tool definitions
└── AppState.tsx       - State management

Tier 2: Important (Core functionality)
├── toolOrchestration.ts - Tool execution pipeline
├── toolExecution.ts     - Individual tool execution
├── context.ts           - Context building
├── bridgeMain.ts        - Bridge coordinator
└── store.ts            - Simple state store

Tier 3: Significant (Important subsystems)
├── services/api/claude.ts    - API client
├── services/compact/        - Context compaction
├── services/mcp/client.ts  - MCP integration
├── ink/reconciler.ts       - React reconciler
└── commands.ts             - Command registry
```

## Tier 1: Critical Files

### 1. main.tsx (~4600 lines)

**Purpose**: Application entry point and bootstrap

**Why Critical**: This is where Claude Code starts executing. It:
- Initializes all modules
- Sets up the command-line interface
- Launches the REPL
- Manages startup sequence

**Key Sections**:

```typescript
// 1. PRE-IMPORT OPTIMIZATION (~50 lines)
// Runs before imports for performance
profileCheckpoint('main_tsx_entry')
startMdmRawRead()
startKeychainPrefetch()

// 2. IMPORTS (~200 lines)
// All module imports

// 3. MAIN FUNCTION (~500 lines)
// Command registration
// Startup sequence
// REPL launch

// 4. HELPER FUNCTIONS (~2000 lines)
// Various setup and utility functions

// 5. COMMAND HANDLERS (~2000 lines)
// 100+ command implementations
```

**Key Insight**: The first ~50 lines before imports are crucial for startup performance. Background I/O is started before heavy module loading begins.

---

### 2. query.ts (~1700 lines)

**Purpose**: High-level query orchestration

**Why Critical**: This is the "brain" — it coordinates the entire AI interaction from input to output.

**Key Functions**:

```typescript
// Main query function
export async function query(
  params: QueryParams
): AsyncGenerator<QueryEvent>

// Build conversation messages
async function buildConversationMessages(
  params: QueryParams
): Promise<Message[]>

// Add memory files to context
async function addMemoryFiles(
  context: QueryContext
): Promise<QueryContext>

// Handle tool execution results
async function handleToolResults(
  results: ToolResult[]
): Promise<void>
```

**Key Insight**: The query function is an async generator — it yields events as they happen, enabling real-time streaming to the UI.

---

### 3. QueryEngine.ts (~1200 lines)

**Purpose**: Low-level streaming implementation

**Why Critical**: This is where the rubber meets the road — it handles the actual API communication and streaming.

**Key Class**:

```typescript
export class QueryEngine {
  private client: Anthropic
  private messages: Message[]
  private tools: Tool[]
  
  async *stream(): AsyncGenerator<StreamEvent> {
    // Stream events from API
    const stream = await this.client.messages.stream({
      model: this.model,
      messages: this.messages,
      tools: this.tools,
    })
    
    for await (const event of stream) {
      yield this.processEvent(event)
    }
  }
}
```

**Key Insight**: This class handles the raw streaming from Anthropic's API, processing events as they arrive and converting them to Claude Code's internal event format.

---

### 4. Tool.ts (~700 lines)

**Purpose**: Tool definitions and interfaces

**Why Critical**: Defines what tools are and how they work.

**Key Interfaces**:

```typescript
// The Tool interface (all tools implement this)
export interface Tool {
  name: string
  description: string
  inputSchema: ToolInputJSONSchema
  permission?: ToolPermission
  execute(
    input: unknown,
    context: ToolUseContext
  ): AsyncGenerator<ToolProgress> | Promise<ToolResult>
}

// Tool use context (injected into tools)
export interface ToolUseContext {
  canUseTool: CanUseToolFn
  getAppState: () => AppState
  setAppState: SetAppState
  abortController: AbortController
  // ...
}

// Tool input schema type
export type ToolInputJSONSchema = {
  type: 'object'
  properties?: Record<string, unknown>
  required?: string[]
}
```

**Key Insight**: Tools are interface-based, making them interchangeable and testable.

---

### 5. AppState.tsx (~200 lines)

**Purpose**: Global state management

**Why Critical**: Holds all the state that any part of the application might need.

**Key Exports**:

```typescript
// App state type
export interface AppState {
  messages: Message[]
  tasks: TaskState[]
  permissions: PermissionState
  teammates: Teammate[]
  sessionId: string
  // ... more fields
}

// React provider
export function AppStateProvider({ children, initialState }) {
  const [store] = useState(() => createStore(initialState))
  return (
    <AppStoreContext.Provider value={store}>
      {children}
    </AppStoreContext.Provider>
  )
}

// Custom hook
export function useAppState(): AppState {
  const store = useContext(AppStoreContext)
  const [state, setState] = useState(store!.getState)
  
  useEffect(() => {
    return store!.subscribe(() => setState(store!.getState()))
  }, [store])
  
  return state
}
```

**Key Insight**: State is accessed through React context, but the underlying store is a simple reactive pattern (~35 lines).

---

## Tier 2: Important Files

### 6. toolOrchestration.ts (~200 lines)

**Purpose**: Orchestrates multiple tool calls

**Why Important**: Determines how tools are executed — concurrently or serially.

**Key Function**:

```typescript
export async function* runTools(
  toolUseMessages: ToolUseBlock[],
  assistantMessages: AssistantMessage[],
  canUseTool: CanUseToolFn,
  toolUseContext: ToolUseContext,
): AsyncGenerator<MessageUpdate> {
  // Partition tools by concurrency safety
  const { safeTools, unsafeTools } = partitionBySafety(toolUseMessages)
  
  // Run read-only tools concurrently
  if (safeTools.length > 0) {
    yield* runToolsConcurrently(safeTools, toolUseContext)
  }
  
  // Run stateful tools serially
  if (unsafeTools.length > 0) {
    yield* runToolsSerially(unsafeTools, toolUseContext)
  }
}
```

**Key Insight**: Read-only tools (like FileReadTool, GlobTool) can run in parallel, but stateful tools (like BashTool, FileWriteTool) must run one at a time.

---

### 7. toolExecution.ts (~1700 lines)

**Purpose**: Executes individual tool calls

**Why Important**: Handles the actual tool execution, permission checks, and progress streaming.

**Key Function**:

```typescript
export async function* runToolUse(
  toolUse: ToolUseBlock,
  context: ToolUseContext,
): AsyncGenerator<MessageUpdate> {
  // 1. Find the tool
  const tool = findToolByName(toolUse.name)
  
  // 2. Check permissions
  const canUse = await context.canUseTool(tool.name)
  if (!canUse) {
    yield* requestPermission(tool, context)
    return
  }
  
  // 3. Execute with progress
  for await (const progress of tool.execute(toolUse.input, context)) {
    yield { progress }
  }
}
```

---

### 8. context.ts (~200 lines)

**Purpose**: Builds the system context

**Why Important**: Provides Claude with relevant project information.

**Key Functions**:

```typescript
export async function getSystemContext(): Promise<string> {
  const [gitStatus, branch, projectInfo] = await Promise.all([
    getGitStatus(),
    getBranch(),
    getProjectInfo(),
  ])
  
  return formatContext(gitStatus, branch, projectInfo)
}

export async function getMemoryFiles(): Promise<MemoryFile[]> {
  // Read .claude/memory.md and other memory files
}
```

---

### 9. bridgeMain.ts (~3000 lines)

**Purpose**: Bridge coordinator

**Why Important**: Manages remote sessions and cross-process communication.

**Key Functions**:

```typescript
export class BridgeMain {
  private sessions: Map<string, SessionHandle>
  
  async connect(config: BridgeConfig): Promise<void> {
    // Establish connection to remote bridge
  }
  
  async spawnSession(opts: SessionOpts): Promise<SessionHandle> {
    // Spawn a new session
  }
  
  async sendMessage(sessionId: string, msg: Message): Promise<void> {
    // Send message to session
  }
}
```

---

### 10. store.ts (~35 lines)

**Purpose**: Simple reactive store

**Why Important**: Powers the entire state management system with minimal code.

**Complete Implementation**:

```typescript
type Listener = () => void

export type Store<T> = {
  getState: () => T
  setState: (updater: (prev: T) => T) => void
  subscribe: (listener: Listener) => () => void
}

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

**Key Insight**: ~35 lines provides all the reactivity needed. Uses `Object.is` to skip unnecessary updates and `Set` for efficient listeners.

---

## Tier 3: Significant Files

### 11. services/api/claude.ts (~3400 lines)

**Purpose**: Anthropic API client

**Key Functions**:
- `createMessage()` — Send a regular message
- `streamMessage()` — Send a streaming message
- `handleError()` — Error classification and retry logic

---

### 12. services/compact/compact.ts (~1600 lines)

**Purpose**: Context compaction

**Key Functions**:
- `compactMessages()` — Summarize older messages
- `findCompactBoundary()` — Find where to cut
- `summarize()` — Ask Claude to summarize

---

### 13. services/mcp/client.ts (~3300 lines)

**Purpose**: MCP client implementation

**Key Functions**:
- `connect()` — Connect to MCP server
- `listTools()` — Get available tools
- `callTool()` — Call an MCP tool

---

### 14. ink/reconciler.ts (~400 lines)

**Purpose**: React reconciler for terminal

**Key Functions**:
- `reconcile()` — Diff and update tree
- `commitUpdate()` — Apply changes to terminal

---

### 15. commands.ts (~25000 lines!)

**Purpose**: Command registry

**Why Important**: Contains all 100+ slash commands.

**Key Structure**:

```typescript
export interface Command {
  name: string
  description: string
  execute(args: string[]): Promise<void>
}

export function getCommands(): Command[] {
  return [
    // Edit commands
    { name: 'edit', execute: handleEdit },
    { name: 'new', execute: handleNew },
    
    // Git commands
    { name: 'commit', execute: handleCommit },
    { name: 'branch', execute: handleBranch },
    
    // ... 100+ more
  ]
}
```

---

## File Statistics

| File | Lines | Purpose |
|------|-------|---------|
| main.tsx | ~4600 | Entry point |
| commands.ts | ~25000 | All commands |
| services/api/claude.ts | ~3400 | API client |
| services/mcp/client.ts | ~3300 | MCP client |
| bridgeMain.ts | ~3000 | Bridge coordinator |
| toolExecution.ts | ~1700 | Tool execution |
| query.ts | ~1700 | Query orchestration |
| services/compact/compact.ts | ~1600 | Compaction |
| QueryEngine.ts | ~1200 | Streaming |
| ink/reconciler.ts | ~400 | React reconciler |
| Tool.ts | ~700 | Tool definitions |
| AppState.tsx | ~200 | State management |
| context.ts | ~200 | Context building |
| store.ts | ~35 | Simple store |

---

## Summary

The critical files follow a clear pattern:

1. **Entry (main.tsx)** — Bootstrap and initialize
2. **Query (query.ts, QueryEngine.ts)** — Coordinate AI interaction
3. **Tools (Tool.ts, toolOrchestration.ts, toolExecution.ts)** — Execute actions
4. **State (AppState.tsx, store.ts)** — Hold and manage state
5. **Context (context.ts)** — Provide relevant information

Understanding these files is the key to understanding Claude Code.
