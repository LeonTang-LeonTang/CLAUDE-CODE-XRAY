# Architecture Diagram

## High-Level System Architecture

```mermaid
graph TB
    subgraph CLI["CLI Layer (main.tsx, cli/, commands/)"]
        CLI_ENTRY["Entry Point<br/>(main.tsx ~4600 lines)"]
        CLI_REPL["REPL Launcher<br/>(replLauncher.tsx)"]
        CLI_CMDS["Commands<br/>(100+ slash commands)"]
        CLI_ARGS["Argument Parser"]
    end

    subgraph BRIDGE["Bridge Layer (bridge/)"]
        BRIDGE_MAIN["Bridge Main<br/>(bridgeMain.ts)"]
        BRIDGE_API["Bridge API Client<br/>(bridgeApi.ts)"]
        BRIDGE_SESSION["Session Runner<br/>(sessionRunner.ts)"]
        BRIDGE_PERMS["Permission System"]
        BRIDGE_AUTH["JWT Auth & Tokens"]
    end

    subgraph QUERY["Query Engine (query.ts, QueryEngine.ts)"]
        QUERY_MAIN["Query Orchestrator<br/>(query.ts ~1700 lines)"]
        QUERY_STREAM["Streaming Engine<br/>(QueryEngine.ts ~1200 lines)"]
        QUERY_CONTEXT["Context Builder<br/>(context.ts)"]
        QUERY_COMPACT["Compaction System<br/>(services/compact/)"]
    end

    subgraph TOOLS["Tool System (tools/)"]
        TOOL_ORCH["Tool Orchestration<br/>(toolOrchestration.ts)"]
        TOOL_EXEC["Tool Executor<br/>(toolExecution.ts)"]
        
        subgraph TOOL_FILE["File Tools"]
            TF_READ["FileReadTool"]
            TF_EDIT["FileEditTool"]
            TF_WRITE["FileWriteTool"]
            TF_GLOB["GlobTool"]
            TF_GREP["GrepTool"]
        end
        
        subgraph TOOL_SHELL["Shell Tools"]
            TS_BASH["BashTool"]
            TS_PWSH["PowerShellTool"]
        end
        
        subgraph TOOL_WEB["Web Tools"]
            TW_SEARCH["WebSearchTool"]
            TW_FETCH["WebFetchTool"]
        end
        
        subgraph TOOL_AGENT["Agent Tools"]
            TA_MAIN["AgentTool"]
            TA_TASK["Task Tools"]
        end
        
        subgraph TOOL_MCP["MCP Integration"]
            TM_CLIENT["MCP Client"]
            TM_WRAP["Tool Wrapper"]
        end
        
        subgraph TOOL_OTHER["Other Tools"]
            TO_CONFIG["ConfigTool"]
            TO_LSP["LSPTool"]
            TO_TODO["TodoWriteTool"]
            TO_SKILL["SkillTool"]
        end
    end

    subgraph STATE["State & Rendering (state/, ink/)"]
        STATE_APP["AppState.tsx<br/>(React Context)"]
        STATE_STORE["Store (Zustand-like)"]
        STATE_HOOKS["Hooks (88+ files)"]
        
        subgraph INK["Ink Terminal UI"]
            INK_RECON["React Reconciler"]
            INK_LAYOUT["Layout Engine (Yoga)"]
            INK_EVENTS["Event System"]
            INK_COMP["Components (Box, Text...)"]
        end
    end

    subgraph API["External APIs"]
        API_ANTHROPIC["Anthropic API"]
        API_MCP["MCP Servers"]
        API_TELEMETRY["Analytics/Telemetry"]
    end

    CLI_ENTRY --> BRIDGE_MAIN
    CLI_REPL --> BRIDGE_MAIN
    CLI_CMDS --> QUERY_MAIN
    CLI_ARGS --> QUERY_MAIN
    
    BRIDGE_MAIN --> QUERY_MAIN
    BRIDGE_API --> API_ANTHROPIC
    BRIDGE_SESSION --> BRIDGE_API
    
    QUERY_MAIN --> QUERY_STREAM
    QUERY_STREAM --> QUERY_CONTEXT
    QUERY_MAIN --> QUERY_COMPACT
    
    QUERY_STREAM --> TOOL_ORCH
    TOOL_ORCH --> TOOL_EXEC
    
    TOOL_EXEC --> TOOL_FILE
    TOOL_EXEC --> TOOL_SHELL
    TOOL_EXEC --> TOOL_WEB
    TOOL_EXEC --> TOOL_AGENT
    TOOL_EXEC --> TOOL_MCP
    TOOL_EXEC --> TOOL_OTHER
    
    TOOL_MCP --> API_MCP
    TM_CLIENT --> API_MCP
    
    TOOL_ORCH --> STATE_APP
    QUERY_STREAM --> STATE_APP
    BRIDGE_MAIN --> STATE_APP
    
    STATE_APP --> STATE_STORE
    STATE_APP --> STATE_HOOKS
    STATE_APP --> INK_RECON
    
    INK_RECON --> INK_LAYOUT
    INK_RECON --> INK_EVENTS
    INK_RECON --> INK_COMP
```

## Query Loop Architecture

```mermaid
sequenceDiagram
    participant User as User Input
    participant CLI as CLI Layer
    participant Q as Query Engine
    participant API as Anthropic API
    participant T as Tool System
    participant S as State/UI
    
    User->>CLI: Type message
    CLI->>Q: processUserInput()
    Q->>Q: Build context<br/>(git, files, memories)
    Q->>API: Send messages
    
    loop Streaming Response
        API-->>Q: Stream events
        Q->>Q: Handle text delta
        Q->>S: Update UI (streaming)
        
        alt Tool Call Detected
            Q->>T: Execute tool()
            T->>S: Stream progress
            S-->>User: Show progress
            T-->>Q: Tool result
            Q->>Q: Add to conversation
            Q->>API: Continue API call
        end
    end
    
    API-->>Q: Stop reason reached
    Q-->>S: Final update
    S-->>User: Display complete
```

## Tool Execution Flow

```mermaid
flowchart LR
    subgraph Detection["1. Tool Detection"]
        D_STREAM["Streaming Response"]
        D_PARSE["Parse Tool Call"]
        D_BLOCK["ToolUseBlock"]
    end
    
    subgraph Permission["2. Permission Check"]
        P_CHECK["CanUseTool()"]
        P_STATE["Permission State"]
        P_ASK{"User Response?"}
        P_APPROVE[Approved]
        P_DENY[Denied]
    end
    
    subgraph Execution["3. Tool Execution"]
        E_FIND["findToolByName()"]
        E_PARTITION["Partition by Safety"]
        E_CONCURRENT["Read-only: Concurrent"]
        E_SERIAL["Stateful: Serial"]
        E_STREAM["Stream Progress"]
        E_RESULT["Tool Result"]
    end
    
    subgraph Result["4. Result Processing"]
        R_ADD["Add to Messages"]
        R_MAYBE_COMPACT["Check Compaction"]
        R_LOOP["Loop or Exit"]
    end
    
    D_STREAM --> D_PARSE --> D_BLOCK
    D_BLOCK --> P_CHECK --> P_STATE
    P_STATE --> P_ASK
    P_ASK -->|Yes| P_APPROVE
    P_ASK -->|No| P_DENY
    P_APPROVE --> E_FIND
    E_FIND --> E_PARTITION
    E_PARTITION --> E_CONCURRENT
    E_PARTITION --> E_SERIAL
    E_CONCURRENT --> E_STREAM
    E_SERIAL --> E_STREAM
    E_STREAM --> E_RESULT
    E_RESULT --> R_ADD
    R_ADD --> R_MAYBE_COMPACT
    R_MAYBE_COMPACT --> R_LOOP
```

## State Management Architecture

```mermaid
graph TB
    subgraph AppState["AppState (Global State)"]
        AS_TASKS["tasks: TaskState[]"]
        AS_PERMS["permissions: PermissionState"]
        AS_MSGS["messages: Message[]"]
        AS_TEAM["teammates: TeammateState[]"]
        AS_SESSION["session: SessionState"]
    end
    
    subgraph Store["Store Implementation"]
        S_CREATE["createStore()"]
        S_STATE["State<T>"]
        S_SUBSCRIBE["Subscribe/Notify"]
    end
    
    subgraph React["React Integration"]
        R_PROVIDER["AppStateProvider"]
        R_CONTEXT["AppStoreContext"]
        R_HOOKS["useAppState()"]
    end
    
    subgraph Hooks["Specialized Hooks"]
        H_CANUSE["useCanUseTool()"]
        H_IDE["useIdeSelection()"]
        H_PERMS["usePermission()"]
    end
    
    S_CREATE --> S_STATE
    S_STATE --> S_SUBSCRIBE
    R_PROVIDER --> R_CONTEXT
    R_CONTEXT --> R_HOOKS
    R_HOOKS --> H_CANUSE
    R_HOOKS --> H_IDE
    R_HOOKS --> H_PERMS
    
    AppState --> Store
    Store --> React
    React --> Hooks
```

## Terminal Rendering (Ink System)

```mermaid
graph TB
    subgraph React["React Components"]
        RC_APP["<App>"]
        RC_BOX["<Box>"]
        RC_TEXT["<Text>"]
        RC_BTN["<Button>"]
    end
    
    subgraph Reconciler["Custom Reconciler"]
        REC_ON["onCompleteWork()"]
        REC_DIFF["Diff Algorithm"]
        REC_QUEUE["Update Queue"]
    end
    
    subgraph Layout["Layout Engine"]
        L_YOGA["Yoga Layout"]
        L_BOX["Box Model Calc"]
        L_FLEX["Flex Layout"]
    end
    
    subgraph Output["Output Generation"]
        O_ANSI["ANSI Codes"]
        O_CSI["CSI Sequences"]
        O_ESC["Escape Sequences"]
        O_SGR["SGR Codes"]
    end
    
    subgraph Terminal["Terminal"]
        T_OUT["stdout write"]
        T_IN["stdin read"]
        T_EVENTS["Events"]
    end
    
    React --> Reconciler
    Reconciler --> Layout
    Layout --> Output
    Output --> Terminal
    Terminal --> Reconciler
```

## MCP (Model Context Protocol) Integration

```mermaid
graph TB
    subgraph ClaudeCode["Claude Code"]
        MC_CLIENT["MCP Client (client.ts)"]
        MC_TOOL["MCPTool Wrapper"]
        MC_RES["Resource Tools"]
    end
    
    subgraph Transport["Transport Layer"]
        T_STDIO["stdio Transport"]
        T_SSE["SSE Transport"]
        T_HTTP["Streamable HTTP"]
    end
    
    subgraph Protocol["MCP Protocol"]
        P_INIT["initialize"]
        P_LIST["list_tools"]
        P_CALL["call_tool"]
        P_RES["list_resources"]
        P_READ["read_resource"]
    end
    
    subgraph Server["MCP Server"]
        S_HANDLER["Tool Handler"]
        S_RES["Resource Handler"]
        S_AUTH["Auth Handler"]
    end
    
    MC_CLIENT --> T_STDIO
    MC_CLIENT --> T_SSE
    MC_CLIENT --> T_HTTP
    
    T_STDIO --> P_INIT
    T_SSE --> P_INIT
    T_HTTP --> P_INIT
    
    P_INIT --> S_HANDLER
    P_LIST --> S_HANDLER
    P_CALL --> S_HANDLER
    P_RES --> S_RES
    P_READ --> S_RES
```

## File Structure Overview

```
claude-code-xray/
├── main.tsx                          # Main entry (~4600 lines)
├── query.ts                          # Query orchestration (~1700 lines)
├── QueryEngine.ts                    # Streaming engine (~1200 lines)
├── Tool.ts                           # Tool definitions (~700 lines)
├── tools.ts                          # Tool registry (~400 lines)
├── context.ts                        # Context gathering (~200 lines)
│
├── cli/                              # CLI arguments
├── commands/                         # 100+ slash commands
├── bridge/                           # Session management
│   ├── bridgeMain.ts                 # Bridge coordinator
│   ├── bridgeApi.ts                  # API client
│   ├── sessionRunner.ts              # Session spawning
│   └── types.ts                      # Shared types
│
├── services/                         # Business logic services
│   ├── api/                          # API calls
│   ├── compact/                      # Context compaction
│   ├── mcp/                         # MCP integration
│   └── tools/                       # Tool execution
│
├── tools/                            # Tool implementations
│   ├── BashTool/                     # Shell execution
│   ├── FileReadTool/                 # File reading
│   ├── FileEditTool/                 # File editing
│   ├── MCPTool/                      # MCP wrapper
│   └── AgentTool/                    # Sub-agents
│
├── state/                            # State management
│   ├── AppState.tsx                  # React context
│   └── store.ts                      # Store implementation
│
├── ink/                              # Terminal UI
│   ├── reconciler.ts                 # React reconciler
│   ├── layout/                       # Layout engine
│   └── components/                   # UI components
│
├── hooks/                            # React hooks (88+)
├── utils/                            # Utilities (332 files)
└── types/                            # TypeScript types
```

## Communication Patterns

```mermaid
graph LR
    subgraph IPC["Inter-Process Communication"]
        IPC_BRIDGE["Bridge ↔ Session"]
        IPC_MAIN["Main ↔ Worker"]
        IPC_STDIN["stdin/stdout"]
    end
    
    subgraph Async["Async/Await Pattern"]
        ASYNC_QUERY["Query (async)"]
        ASYNC_TOOL["Tool Execution"]
        ASYNC_STREAM["Streaming"]
    end
    
    subgraph Events["Event System"]
        EV_HOOKS["Hook System"]
        EV_STATE["State Updates"]
        EV_TERMINAL["Terminal Events"]
    end
    
    IPC --> Async
    Async --> Events
```

## Session Lifecycle

```mermaid
stateDiagram-v2
    [*] --> Init: main.tsx entry
    
    Init --> LoadingConfig: Parse args
    LoadingConfig --> LoadingState: Load settings
    LoadingState --> InitMCP: Setup MCP servers
    
    InitMCP --> Ready: MCP connected
    Ready --> QueryLoop: User input
    
    QueryLoop --> ToolExecution: Tool call
    ToolExecution --> QueryLoop: Tool done
    
    QueryLoop --> Compact: Context full
    Compact --> QueryLoop: Compaction done
    
    QueryLoop --> Error: API error
    Error --> Retry: Backoff
    Retry --> QueryLoop: Retry
    
    QueryLoop --> [*]: Exit command
```
