# Entry Point: Where It All Starts

This document provides a detailed walkthrough of Claude Code's entry point and critical starting files, explaining how the application bootstraps and initializes all its systems.

## The Starting Line (main.tsx)

**File**: `main.tsx` (~4600 lines)

### The Very First Lines

```typescript
// These MUST run before all other imports
// Reason: Performance optimization - start background work early

// Profile marker - marks when we entered main.tsx
import { profileCheckpoint, profileReport } from './utils/startupProfiler.js'
profileCheckpoint('main_tsx_entry')

// Start reading MDM (Mobile Device Management) settings
// This fires off subprocesses that run in parallel with imports
import { startMdmRawRead } from './utils/settings/mdm/rawRead.js'
startMdmRawRead()

// Prefetch keychain tokens while imports are happening
// Without this: 65ms sequential reads on every startup
// With this: Reads happen in parallel with module loading
import { startKeychainPrefetch } from './utils/secureStorage/keychainPrefetch.js'
startKeychainPrefetch()
```

**💡 Insight**: This pattern of starting background work before heavy imports is crucial for startup performance. By the time all modules are loaded (~135ms), the keychain data is already ready.

### Feature Flags

Claude Code uses feature flags for dead code elimination:

```typescript
import { feature } from 'bun:bundle'

// Dead code elimination: these modules are removed from builds
// where the feature is not enabled
const coordinatorModeModule = feature('COORDINATOR_MODE')
  ? require('./coordinator/coordinatorMode.js')
  : null

const assistantModule = feature('KAIROS')
  ? require('./assistant/index.js')
  : null
```

**💡 Insight**: This allows Claude Code to ship different builds (internal, public, enterprise) with different features, while keeping each binary lean.

### Command Registration

```typescript
// After imports, register all commands
async function main() {
  const program = new Command()
  
  // Set up version
  program
    .name('claude')
    .version('1.x.x')
  
  // Main command - launches the REPL
  program
    .command('claude')
    .description('Start Claude Code')
    .option('-p, --print <prompt>', 'Print response to prompt')
    .option('-m, --model <model>', 'Model to use')
    .action(async (options) => {
      await launchRepl(options)
    })
  
  // Subcommands
  program
    .command('config')
    .description('Configure Claude Code')
    .action(() => handleConfigCommand())
  
  // ... 100+ more commands
  
  await program.parseAsync(process.argv)
}

main()
```

### Startup Sequence

```typescript
async function launchRepl(options: LaunchOptions): Promise<void> {
  // Phase 1: Pre-flight checks
  await checkAPIKey()
  await checkVersion()
  
  // Phase 2: Initialize state
  const initialState = createInitialState()
  
  // Phase 3: Setup telemetry
  await initializeTelemetry()
  
  // Phase 4: Connect to MCP servers
  await setupMCPServers()
  
  // Phase 5: Load session
  await loadOrCreateSession()
  
  // Phase 6: Launch REPL
  await launchInteractiveREPL(initialState)
}

async function checkAPIKey(): Promise<void> {
  const apiKey = process.env.ANTHROPIC_API_KEY || await getStoredAPIKey()
  
  if (!apiKey) {
    // Show API key setup screen
    await showAPIKeyPrompt()
    process.exit(1)
  }
  
  // Store for API client
  setAPIKey(apiKey)
}
```

## The REPL Launcher (replLauncher.tsx)

**File**: `replLauncher.tsx`

### Launching the Terminal UI

```typescript
export async function launchRepl(options: ReplOptions): Promise<void> {
  // Create the React app
  const app = createElement(App, {
    initialState: getDefaultAppState(),
    onExit: handleExit,
  })
  
  // Render with Ink (React-to-terminal renderer)
  const { unmount, waitUntilExit } = render(app)
  
  // Setup signal handlers
  setupSignalHandlers(unmount)
  
  // Wait for REPL to exit
  await waitUntilExit()
  
  // Cleanup
  await cleanup()
  process.exit(0)
}
```

## The App Component (ink/components/App.tsx)

**File**: `ink/components/App.tsx`

### Main Application Component

```typescript
// The root React component for the terminal UI
export function App({ initialState, onExit }) {
  // State management
  const [state, setState] = useState(initialState)
  const [mode, setMode] = useState('idle')
  
  // App state provider
  const store = useMemo(
    () => createStore(initialState, handleStateChange),
    []
  )
  
  return (
    <AppStoreContext.Provider value={store}>
      <TerminalSizeProvider>
        <KeyboardProvider>
          <Box flexDirection="column" padding={1}>
            {/* Header */}
            <Header state={state} />
            
            {/* Main content area */}
            <MainContent state={state} mode={mode} />
            
            {/* Input area */}
            <InputArea 
              onSubmit={handleSubmit}
              disabled={mode === 'loading'}
            />
          </Box>
        </KeyboardProvider>
      </TerminalSizeProvider>
    </AppStoreContext.Provider>
  )
}
```

## State Initialization (state/AppStateStore.ts)

**File**: `state/AppStateStore.ts`

### Initial State

```typescript
export interface AppState {
  // Conversation
  messages: Message[]
  
  // Tasks (shell commands, sub-agents)
  tasks: TaskState[]
  
  // Permissions
  permissions: PermissionState
  permissionHistory: PermissionRecord[]
  
  // Teammates (multi-agent)
  teammates: Teammate[]
  
  // Session info
  sessionId: string
  sessionStarted: Date
  
  // UI state
  isLoading: boolean
  activeTask: string | null
  inputValue: string
  
  // Settings
  settings: Settings
  
  // MCP servers
  mcpServers: MCPServerState[]
}

export function getDefaultAppState(): AppState {
  return {
    messages: [],
    tasks: [],
    permissions: { status: 'ready' },
    permissionHistory: [],
    teammates: [],
    sessionId: generateSessionId(),
    sessionStarted: new Date(),
    isLoading: false,
    activeTask: null,
    inputValue: '',
    settings: getDefaultSettings(),
    mcpServers: [],
  }
}
```

## The Store Implementation (state/store.ts)

**File**: `state/store.ts`

### Simple Reactive Store

```typescript
type Listener = () => void

export type Store<T> = {
  getState: () => T
  setState: (updater: (prev: T) => T) => void
  subscribe: (listener: Listener) => () => void
}

export function createStore<T>(
  initialState: T,
  onChange?: (args: { newState: T; oldState: T }) => void
): Store<T> {
  let state = initialState
  const listeners = new Set<Listener>()

  return {
    getState: () => state,
    
    setState: (updater: (prev: T) => T) => {
      const prev = state
      const next = updater(prev)
      
      // Skip if no change (optimization)
      if (Object.is(next, prev)) return
      
      state = next
      onChange?.({ newState: next, oldState: prev })
      
      // Notify all listeners
      for (const listener of listeners) {
        listener()
      }
    },
    
    subscribe: (listener: Listener) => {
      listeners.add(listener)
      return () => listeners.delete(listener)
    },
  }
}
```

**💡 Insight**: This is only ~35 lines but provides all the reactivity Claude Code needs. It's inspired by Zustand but much simpler. The key insight is using `Object.is(next, prev)` to skip unnecessary updates.

## Query Initialization (query.ts)

**File**: `query.ts` (~1700 lines)

### Query Function

```typescript
export async function query(
  params: QueryParams
): Promise<AsyncGenerator<QueryEvent>> {
  // 1. Validate and normalize input
  const normalized = normalizeInput(params.input)
  
  // 2. Build conversation context
  const context = await buildQueryContext(normalized)
  
  // 3. Add memory files
  const withMemory = await addMemoryFiles(context)
  
  // 4. Get tools
  const tools = await getTools(params.appState)
  
  // 5. Create query engine
  const engine = new QueryEngine({
    messages: withMemory,
    model: params.model,
    tools,
    systemPrompt: buildSystemPrompt(params),
  })
  
  // 6. Execute query
  yield* engine.stream()
}
```

## The Query Engine (QueryEngine.ts)

**File**: `QueryEngine.ts` (~1200 lines)

### Streaming Implementation

```typescript
export class QueryEngine {
  private client: Anthropic
  private messages: Message[]
  private tools: Tool[]
  
  constructor(params: QueryEngineParams) {
    this.client = new Anthropic({ apiKey: getAPIKey() })
    this.messages = params.messages
    this.tools = params.tools
  }
  
  async *stream(): AsyncGenerator<StreamEvent> {
    // Build API request
    const request = this.buildRequest()
    
    // Stream response
    const stream = await this.client.messages.stream(request)
    
    for await (const event of stream) {
      yield this.processEvent(event)
    }
  }
  
  private buildRequest(): MessageStreamParams {
    return {
      model: this.model,
      max_tokens: 8192,
      messages: this.messages.map(toAPIMessage),
      tools: this.tools.map(toAPISchema),
    }
  }
  
  private processEvent(event: RawStreamEvent): StreamEvent {
    // Handle different event types
    // ...
  }
}
```

## CLI Argument Parsing

**File**: `cli/handlers/args.ts`

```typescript
interface CLIArgs {
  // Mode
  print?: string        // -p, --print <prompt>
  resume?: string       // --resume <session>
  
  // Model
  model?: string        // --model <model>
  
  // Context
  context?: string      // --context <file>
  
  // Output
  outputFormat?: 'text' | 'json'  // --output-format
}

// Parse arguments
export function parseArgs(argv: string[]): CLIArgs {
  const args: CLIArgs = {}
  const tokens = argv.slice(2)  // Skip 'claude'
  
  for (let i = 0; i < tokens.length; i++) {
    const token = tokens[i]
    
    if (token === '-p' || token === '--print') {
      args.print = tokens[++i]
    } else if (token === '--model') {
      args.model = tokens[++i]
    } else if (token === '--resume') {
      args.resume = tokens[++i]
    }
    // ... more parsing
  }
  
  return args
}
```

## Environment Setup

```typescript
// Apply environment variables early
async function applyEnvironment(): Promise<void> {
  // Claude Code settings that can come from environment
  const envConfig = {
    apiKey: process.env.ANTHROPIC_API_KEY,
    model: process.env.ANTHROPIC_MODEL,
    baseURL: process.env.ANTHROPIC_BASE_URL,
    
    // Debug settings
    debug: process.env.CLAUDE_DEBUG === 'true',
    verbose: process.env.CLAUDE_VERBOSE === 'true',
    
    // Feature flags
    enableMcp: process.env.CLAUDE_ENABLE_MCP !== 'false',
  }
  
  // Apply to config
  applyConfig(envConfig)
}
```

## Startup Profiler

**File**: `utils/startupProfiler.ts`

```typescript
interface ProfileCheckpoint {
  name: string
  startTime: number
  endTime?: number
}

const checkpoints: ProfileCheckpoint[] = []

export function profileCheckpoint(name: string): void {
  checkpoints.push({
    name,
    startTime: performance.now(),
  })
}

export function profileReport(): void {
  const report = checkpoints.map((cp, i) => {
    const duration = cp.endTime 
      ? cp.endTime - cp.startTime 
      : performance.now() - cp.startTime
    
    return `${cp.name}: ${duration.toFixed(2)}ms`
  })
  
  console.log('Startup Profile:')
  console.log(report.join('\n'))
}
```

## Exit Handling

```typescript
// Setup cleanup handlers
function setupExitHandlers(): void {
  const cleanup = async () => {
    console.log('\nCleaning up...')
    
    // Abort running queries
    abortController.abort()
    
    // Save session
    await saveSession()
    
    // Close connections
    await closeConnections()
    
    process.exit(0)
  }
  
  process.on('SIGINT', cleanup)
  process.on('SIGTERM', cleanup)
  
  process.on('uncaughtException', (error) => {
    console.error('Uncaught exception:', error)
    cleanup()
  })
  
  process.on('unhandledRejection', (reason) => {
    console.error('Unhandled rejection:', reason)
  })
}
```

## Summary

The entry point demonstrates several key patterns:

1. **Prefetch optimization** — Start I/O before heavy imports
2. **Feature flags** — Dead code elimination for lean builds
3. **Simple state** — Minimal reactive store (~35 lines)
4. **Async generators** — Streaming throughout
5. **Profile checkpoints** — Built-in performance tracking
6. **Graceful shutdown** — Cleanup on exit signals
