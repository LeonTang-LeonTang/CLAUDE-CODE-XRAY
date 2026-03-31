# Design Patterns: The Blueprint of Claude Code

This document catalogs the design patterns used throughout Claude Code's codebase, explaining how they're applied and why they work.

## Pattern 1: Async Generator Pipeline

**Problem**: Need to stream data while processing tool calls that can yield multiple progress updates.

**Solution**: Use async generators throughout the entire flow.

```typescript
// Query engine streams events
async function* query(params): AsyncGenerator<QueryEvent> {
  for await (const event of engine.stream()) {
    if (event.type === 'tool_call') {
      // Tools also yield progress
      for await (const result of executeTool(event)) {
        yield { type: 'tool_progress', ...result }
      }
    } else {
      yield event
    }
  }
}

// Benefits:
// 1. Memory efficient - no buffering entire responses
// 2. Real-time - UI updates as data arrives
// 3. Backpressure - consumer controls pace
```

**Usage in Claude Code**:
- `query.ts` — Yields events as AI responds
- `toolOrchestration.ts` — Yields tool progress
- `toolExecution.ts` — Individual tool streaming
- `QueryEngine.ts` — API response streaming

---

## Pattern 2: Simple Reactive Store

**Problem**: Need shared state that components can subscribe to, without the complexity of Redux.

**Solution**: A minimal reactive store (~35 lines).

```typescript
type Listener = () => void

export function createStore<T>(initialState: T): Store<T> {
  let state = initialState
  const listeners = new Set<Listener>()

  return {
    getState: () => state,
    
    setState: (updater) => {
      const next = updater(state)
      if (Object.is(next, state)) return  // Skip if unchanged
      state = next
      listeners.forEach(l => l())  // Notify all
    },
    
    subscribe: (listener) => {
      listeners.add(listener)
      return () => listeners.delete(listener)  // Unsubscribe
    },
  }
}
```

**Benefits**:
- ~35 lines vs hundreds for Redux
- Type-safe with TypeScript
- Efficient with `Set` for listeners
- Uses `Object.is` to skip no-op updates

**Usage in Claude Code**: `state/store.ts` — Powers the entire state system.

---

## Pattern 3: Dependency Injection via Context

**Problem**: Need to pass dependencies to tools without tight coupling.

**Solution**: Pass context objects to functions.

```typescript
interface ToolUseContext {
  canUseTool: CanUseToolFn
  getAppState: () => AppState
  setAppState: SetAppState
  abortController: AbortController
  cwd: string
}

// Tool receives everything it needs
class BashTool {
  async execute(input, context: ToolUseContext) {
    // Can access state
    const state = context.getAppState()
    
    // Can update state
    context.setAppState(s => ({ ...s, tasks: [...s.tasks, task] }))
    
    // Can check permissions
    const allowed = await context.canUseTool('BashTool')
  }
}

// Query engine provides the context
const context: ToolUseContext = {
  canUseTool,
  getAppState,
  setAppState,
  abortController,
  cwd,
}
```

**Benefits**:
- Tools are testable (mock the context)
- Tools don't need to know about global state
- Flexible — context can be different in different scenarios

---

## Pattern 4: Feature Flags for DCE

**Problem**: Different builds need different features, but don't want unused code in bundles.

**Solution**: Feature flags with conditional requires.

```typescript
import { feature } from 'bun:bundle'

// Only included in builds with KAIROS feature
const assistantModule = feature('KAIROS')
  ? require('./assistant/index.js')
  : null

// These lines are completely removed by Bun's tree-shaker
// in builds without the feature
if (assistantModule) {
  await assistantModule.initialize()
}
```

**Benefits**:
- Smaller production binaries
- Clean feature branches
- Easy to enable/disable features
- No runtime feature checks overhead (DCE removes dead code)

**Usage in Claude Code**: `main.tsx` — Coordinator mode, Kairos mode, agent triggers.

---

## Pattern 5: Lazy Module Loading

**Problem**: Circular dependencies between modules.

**Solution**: Lazy `require()` inside functions.

```typescript
// Instead of:
import { query } from './query.js'  // Circular!

// Lazy require in bridgeMain.ts:
async function runQuery() {
  // Only import when function is called, not when module loads
  const { query } = await import('./query.js')
  return query(params)
}
```

**Benefits**:
- Breaks circular dependencies
- Modules load in correct order when actually used
- No need for complex dependency ordering

**Usage in Claude Code**: `main.tsx` — Lazy loading of teammate utils.

---

## Pattern 6: Pipeline Pattern

**Problem**: Need to process data through multiple stages.

**Solution**: Chain async generators.

```typescript
async function* processUserInput(input): AsyncGenerator<ProcessedInput> {
  // Stage 1: Parse
  const parsed = yield* parseInput(input)
  
  // Stage 2: Validate
  const validated = yield* validateInput(parsed)
  
  // Stage 3: Transform
  const transformed = yield* transformInput(validated)
  
  return transformed
}

// Usage:
for await (const step of processUserInput(rawInput)) {
  console.log('Processing:', step)
}
```

**Benefits**:
- Composable stages
- Each stage can be tested independently
- Easy to add/remove stages
- Streaming throughout

---

## Pattern 7: Strategy Pattern

**Problem**: Need different algorithms for similar operations.

**Solution**: Select algorithm at runtime.

```typescript
interface CompactionStrategy {
  findBoundary(messages: Message[]): number
  summarize(messages: Message[]): Promise<string>
}

// Different strategies
const simpleStrategy: CompactionStrategy = {
  findBoundary: (msgs) => Math.floor(msgs.length / 2),
  summarize: async (msgs) => 'Summary...',
}

const smartStrategy: CompactionStrategy = {
  findBoundary: (msgs) => findSemanticBoundary(msgs),
  summarize: async (msgs) => await aiSummarize(msgs),
}

// Select at runtime
const strategy = useSmartCompaction ? smartStrategy : simpleStrategy
const boundary = strategy.findBoundary(messages)
```

**Benefits**:
- Swappable algorithms
- Easy to add new strategies
- Configuration-driven behavior

---

## Pattern 8: Observer Pattern

**Problem**: UI needs to react to state changes.

**Solution**: Subscribe to store changes.

```typescript
// Subscribe in React component
function useAppState() {
  const store = useContext(AppStoreContext)
  const [state, setState] = useState(store.getState())
  
  useEffect(() => {
    // Subscribe to changes
    return store.subscribe(() => {
      setState(store.getState())
    })
  }, [store])
  
  return state
}

// Any state change triggers re-render
function updateTask(task) {
  store.setState(s => ({
    ...s,
    tasks: s.tasks.map(t => t.id === task.id ? task : t),
  }))
}
```

**Benefits**:
- Decoupled components
- Centralized state
- Automatic UI updates
- Easy to debug (single source of truth)

---

## Pattern 9: Factory Pattern

**Problem**: Need to create similar but different objects.

**Solution**: Factory functions for tool creation.

```typescript
// Tool factory
function createTool(type: ToolType): Tool {
  switch (type) {
    case 'bash':
      return new BashTool()
    case 'file_read':
      return new FileReadTool()
    case 'file_write':
      return new FileWriteTool()
    default:
      throw new Error(`Unknown tool type: ${type}`)
  }
}

// MCP tool factory
function createMCPTool(server: MCPServer, definition: ToolDef): Tool {
  return {
    name: definition.name,
    description: definition.description,
    inputSchema: definition.inputSchema,
    execute: async (input, ctx) => {
      const result = await server.callTool(definition.name, input)
      return formatResult(result)
    },
  }
}
```

---

## Pattern 10: Command Pattern

**Problem**: Need to encapsulate operations as objects.

**Solution**: Command interface for all operations.

```typescript
interface Command {
  name: string
  execute(args: string[]): Promise<void>
  undo?(): Promise<void>
}

// All commands implement the same interface
const commands: Command[] = [
  {
    name: 'edit',
    execute: async (args) => { /* ... */ },
    undo: async () => { /* revert edit */ },
  },
  {
    name: 'commit',
    execute: async (args) => { /* ... */ },
  },
]

// Execute any command
async function executeCommand(name: string, args: string[]) {
  const cmd = commands.find(c => c.name === name)
  if (!cmd) throw new Error(`Unknown command: ${name}`)
  await cmd.execute(args)
}
```

**Benefits**:
- Uniform interface
- Easy to add new commands
- Commands can be queued, logged, undone

---

## Pattern 11: Memoization

**Problem**: Expensive computations that don't change.

**Solution**: Cache results with memoization.

```typescript
// Memoize git status (called multiple times per query)
export const getGitStatus = memoize(async (): Promise<string | null> => {
  if (!await getIsGit()) return null
  const [branch, status] = await Promise.all([
    execFile('git', ['branch', '--show-current']),
    execFile('git', ['status', '--short']),
  ])
  return formatGitStatus(branch, status)
})

// Usage - only runs once even if called multiple times
const status1 = await getGitStatus()
const status2 = await getGitStatus()  // Returns cached
```

**Usage in Claude Code**:
- `context.ts` — Git status, project info
- `utils/model/model.ts` — Model resolution

---

## Pattern 12: Builder Pattern

**Problem**: Complex object construction with many optional parameters.

**Solution**: Builder class for constructing queries.

```typescript
class QueryBuilder {
  private messages: Message[] = []
  private tools: Tool[] = []
  private model?: string
  private systemPrompt?: string
  
  addMessage(msg: Message): this {
    this.messages.push(msg)
    return this
  }
  
  addTool(tool: Tool): this {
    this.tools.push(tool)
    return this
  }
  
  setModel(model: string): this {
    this.model = model
    return this
  }
  
  build(): QueryParams {
    return {
      messages: this.messages,
      tools: this.tools,
      model: this.model ?? getDefaultModel(),
      systemPrompt: this.systemPrompt ?? getDefaultSystemPrompt(),
    }
  }
}

// Usage
const params = new QueryBuilder()
  .addMessage(userMessage)
  .addTool(FileReadTool)
  .addTool(BashTool)
  .setModel('claude-opus-4-5')
  .build()
```

---

## Pattern 13: Retry with Exponential Backoff

**Problem**: Transient failures need multiple attempts.

**Solution**: Retry with increasing delays.

```typescript
async function withRetry<T>(
  fn: () => Promise<T>,
  maxRetries: number = 3
): Promise<T> {
  for (let attempt = 0; attempt <= maxRetries; attempt++) {
    try {
      return await fn()
    } catch (error) {
      if (attempt === maxRetries || !isRetryable(error)) {
        throw error
      }
      
      // Exponential backoff: 1s, 2s, 4s...
      const delay = 1000 * Math.pow(2, attempt)
      await sleep(delay)
    }
  }
  
  throw new Error('Unreachable')
}
```

**Usage in Claude Code**: `services/api/withRetry.ts` — API calls.

---

## Pattern 14: Event Emitter

**Problem**: Need to notify multiple listeners of events.

**Solution**: Simple event emitter.

```typescript
class EventEmitter {
  private listeners: Map<string, Set<Function>> = new Map()
  
  on(event: string, callback: Function): () => void {
    if (!this.listeners.has(event)) {
      this.listeners.set(event, new Set())
    }
    this.listeners.get(event)!.add(callback)
    
    // Return unsubscribe function
    return () => this.listeners.get(event)!.delete(callback)
  }
  
  emit(event: string, data?: any): void {
    this.listeners.get(event)?.forEach(cb => cb(data))
  }
}

// Usage
const emitter = new EventEmitter()

emitter.on('tool_complete', (tool) => {
  console.log('Tool done:', tool.name)
})

emitter.on('query_complete', (result) => {
  showResult(result)
})
```

**Usage in Claude Code**: Event hooks system.

---

## Pattern 15: Partitioning by Type

**Problem**: Need different handling based on object characteristics.

**Solution**: Partition into groups first, then process.

```typescript
function partitionBySafety(
  tools: ToolUse[]
): { safe: ToolUse[], unsafe: ToolUse[] } {
  return {
    safe: tools.filter(isReadOnlyTool),
    unsafe: tools.filter(t => !isReadOnlyTool(t)),
  }
}

// Then process differently
const { safe, unsafe } = partitionBySafety(allTools)

// Safe tools can run in parallel
await Promise.all(safe.map(t => executeTool(t)))

// Unsafe tools must run serially
for (const tool of unsafe) {
  await executeTool(tool)
}
```

**Benefits**:
- Clear separation of concerns
- Optimal concurrency
- Prevents race conditions

---

## Summary Table

| Pattern | Purpose | Key File |
|---------|---------|----------|
| Async Generator Pipeline | Streaming data | query.ts |
| Simple Reactive Store | State management | state/store.ts |
| Dependency Injection | Decouple dependencies | Tool.ts |
| Feature Flags DCE | Lean builds | main.tsx |
| Lazy Loading | Break cycles | main.tsx |
| Pipeline | Composable stages | toolExecution.ts |
| Strategy | Swappable algorithms | services/compact/ |
| Observer | React to changes | state/AppState.tsx |
| Factory | Create similar objects | tools.ts |
| Command | Encapsulate operations | commands/ |
| Memoization | Cache expensive ops | context.ts |
| Builder | Complex construction | QueryEngine.ts |
| Retry Backoff | Handle transients | services/api/ |
| Event Emitter | Notify listeners | utils/hooks/ |
| Partitioning | Group by type | toolOrchestration.ts |
