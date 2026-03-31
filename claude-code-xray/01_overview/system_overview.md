# System Overview: How Claude Code Thinks

## The Big Picture

Claude Code is a sophisticated **agentic CLI tool** that pairs a developer with Claude (Anthropic's AI) directly in their terminal. But it's far more than a chatbot — it's a full **development environment** with file system access, shell command execution, background task management, and multi-agent collaboration.

Think of it as a **pair programmer that never gets tired** and has access to all your files, your terminal, and your entire codebase.

## Core Philosophy

Claude Code operates on a simple but powerful principle:

> **The AI should be able to do everything a developer can do — edit files, run commands, navigate the codebase — but with the safety of permission prompts and the intelligence to know when to ask for clarification.**

This manifests in several design decisions:

1. **Tool-based interaction** — Claude doesn't just generate text; it calls tools that have real effects
2. **Permission-gated actions** — Destructive or sensitive operations require explicit user approval
3. **Context-aware reasoning** — Claude sees your git status, project structure, and recent changes
4. **Transparent operation** — You can see exactly what Claude is doing via the streaming UI

## The 5-Layer Architecture

Claude Code is organized into 5 distinct layers, each with a specific responsibility:

```
┌────────────────────────────────────────────────────────────────┐
│  LAYER 1: CLI & Commands (cli/, commands/)                     │
│  "The Interface"                                                │
│  Handles user input, slash commands, and program invocation      │
├────────────────────────────────────────────────────────────────┤
│  LAYER 2: Bridge & Session Management (bridge/)                │
│  "The Orchestrator"                                             │
│  Manages remote sessions, permissions, and cross-process comms  │
├────────────────────────────────────────────────────────────────┤
│  LAYER 3: Query Engine (query.ts, QueryEngine.ts)              │
│  "The Brain"                                                     │
│  Coordinates AI interactions, message flow, and context mgmt    │
├────────────────────────────────────────────────────────────────┤
│  LAYER 4: Tool System (tools/)                                  │
│  "The Hands"                                                     │
│  Executes actual operations (files, shells, web, MCP)          │
├────────────────────────────────────────────────────────────────┤
│  LAYER 5: State & Rendering (state/, ink/)                     │
│  "The Nervous System"                                            │
│  Manages UI state, terminal rendering, and permissions          │
└────────────────────────────────────────────────────────────────┘
```

## Layer-by-Layer Breakdown

### Layer 1: CLI & Commands

**Purpose**: Entry point and user interface layer

**Key Components**:
- `main.tsx` — The main entry point (~4600 lines). Handles startup, config loading, and command routing
- `commands/` — 100+ command handlers (slash commands like `/edit`, `/commit`, `/bug`)
- `cli/` — CLI argument parsing and transport setup
- `replLauncher.tsx` — Launches the interactive REPL

**What it does**:
1. Parses command-line arguments
2. Initializes the application state
3. Loads user settings and workspace context
4. Routes to appropriate handler (REPL mode vs command mode)
5. Manages the startup sequence (auth, telemetry, MCP servers)

**Key Insight**: This layer uses a feature-flagged architecture where unused features are removed at build time using `feature('FEATURE_NAME')` checks.

### Layer 2: Bridge & Session Management

**Purpose**: Orchestrates complex operations across process boundaries

**Key Components**:
- `bridgeMain.ts` — Main bridge coordinator with session lifecycle
- `sessionRunner.ts` — Spawns and manages session processes
- `bridgeApi.ts` — API client for remote bridge communication
- `bridgeMessaging.ts` — Message protocol between bridge and sessions
- `permissions.ts` — Permission state machine and callbacks

**What it does**:
1. Manages remote sessions (connecting to cloud-hosted Claude Code)
2. Handles permission escalation (asking user to approve dangerous actions)
3. Coordinates multi-session workflows (teammates, sub-agents)
4. Manages JWT tokens and session authentication
5. Implements backoff and retry logic for network operations

**Key Insight**: The bridge layer enables Claude Code to run on one machine while the actual AI processing happens elsewhere — useful for remote development or team scenarios.

### Layer 3: Query Engine

**Purpose**: The "brain" — coordinates all AI interactions

**Key Components**:
- `query.ts` — The main query orchestration (~1700 lines)
- `QueryEngine.ts` — Streaming query implementation (~1200 lines)
- `context.ts` — System context gathering (git status, project info)
- `services/compact/` — Context compaction and summarization

**What it does**:
1. Takes user input and converts it to API messages
2. Builds system prompts with project context
3. Handles streaming responses from the AI
4. Detects and executes tool calls
5. Manages the message history and context window
6. Triggers context compaction when approaching limits

**The Query Loop** (simplified):
```
User Input → Build Messages → API Call → Stream Response →
  ├── Text Response → Display to User
  └── Tool Call → Execute Tool → Add Result to Messages → Loop
```

**Key Insight**: The query engine implements a **streaming state machine** that handles partial responses, tool calls, errors, and interruptions in real-time.

### Layer 4: Tool System

**Purpose**: The "hands" — actually does things in the real world

**Tool Categories**:

1. **File Operations** (6 tools)
   - `FileReadTool` — Read files with image processing support
   - `FileEditTool` — Edit files using structured diffs
   - `FileWriteTool` — Write/create files
   - `GlobTool` — Find files by pattern
   - `GrepTool` — Search file contents
   - `NotebookEditTool` — Edit Jupyter notebooks

2. **Shell Operations** (2 tools)
   - `BashTool` — Execute shell commands
   - `PowerShellTool` — Execute PowerShell (Windows)

3. **Web Operations** (2 tools)
   - `WebSearchTool` — Search the web
   - `WebFetchTool` — Fetch web pages

4. **Agent Operations** (2 tools)
   - `AgentTool` — Spawn sub-agents
   - `TaskCreateTool/TaskGetTool/TaskUpdateTool/TaskStopTool` — Manage tasks

5. **MCP Tools** (dynamic)
   - `MCPTool` — Wrapper for Model Context Protocol servers
   - `ListMcpResourcesTool` — List MCP resources
   - `ReadMcpResourceTool` — Read MCP resources

6. **Special Tools** (10+ tools)
   - `TodoWriteTool` — Manage TODO lists
   - `LSPTool` — LSP integration for code intelligence
   - `ConfigTool` — Modify Claude Code settings
   - `SkillTool` — Load and run skills
   - And more...

**Tool Execution Pipeline**:
```
Tool Call Detected → Check Permission State →
  ├── UNCONFIRMED → Request Permission → Await Response
  ├── APPROVED → Execute Tool → Stream Progress
  └── DENIED → Report Error → Continue Loop
```

**Key Insight**: Tools are executed through a **generator-based pipeline** (`toolOrchestration.ts`) that supports:
- Concurrent execution of read-only tools
- Serial execution of stateful tools
- Progress streaming for long operations
- Automatic permission escalation

### Layer 5: State & Rendering

**Purpose**: The "nervous system" — manages UI and application state

**Key Components**:
- `state/AppState.tsx` — React context for global application state
- `state/store.ts` — Simple reactive store implementation
- `ink/` — React-to-terminal rendering system
- `hooks/` — 88+ React hooks for various concerns
- `context/` — Additional React contexts

**What it does**:
1. Manages global application state (tasks, messages, permissions)
2. Renders the terminal UI using React components
3. Handles terminal events (keyboard, mouse, resize)
4. Manages component lifecycle and re-rendering
5. Coordinates between different parts of the application

**The Ink System**: A full React renderer that outputs ANSI escape codes instead of HTML:
- Custom reconciler for terminal rendering
- Yoga layout engine for box model calculations
- Event system for keyboard/mouse input
- Animation framework for smooth updates

## How Data Flows

```
┌─────────────────────────────────────────────────────────────────────┐
│                         USER INPUT                                  │
│  "Fix the bug in auth.ts"                                          │
└─────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────┐
│                    CLI LAYER (main.tsx)                             │
│  - Parse arguments                                                  │
│  - Check for slash commands                                         │
│  - Initialize REPL if needed                                        │
└─────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────┐
│              PROCESS USER INPUT (processUserInput.ts)                │
│  - Parse slash commands                                             │
│  - Handle attachments (images, files)                               │
│  - Check for hook-blocked prompts                                   │
│  - Create user message                                              │
└─────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────┐
│              QUERY ENGINE (query.ts)                                  │
│  - Build conversation context                                        │
│  - Add system prompt (project info, git status)                     │
│  - Add memory files (.claude/)                                      │
│  - Send to API                                                      │
└─────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────┐
│              API STREAMING (QueryEngine.ts)                          │
│  - Receive streaming response                                       │
│  - Handle text deltas in real-time                                  │
│  - Detect tool calls                                                │
│  - Stream to UI as they appear                                      │
└─────────────────────────────────────────────────────────────────────┘
                              │
              ┌───────────────┴───────────────┐
              ▼                               ▼
┌──────────────────────────┐    ┌──────────────────────────────┐
│   TEXT RESPONSE          │    │   TOOL CALL                  │
│  Display to user         │    │  ┌────────────────────────┐  │
│  in terminal            │    │  │ Permission Check        │  │
│                         │    │  │ (Is this dangerous?)    │  │
│                         │    │  └────────────────────────┘  │
│                         │    │           │                   │
│                         │    │  ┌────────▼────────┐          │
│                         │    │  │ Execute Tool    │          │
│                         │    │  │ (toolOrchest...)│          │
│                         │    │  └─────────────────┘          │
│                         │    │           │                   │
│                         │    │  ┌────────▼────────┐          │
│                         │    │  │ Stream Progress │          │
│                         │    │  │ Back to Query   │          │
│                         │    │  └─────────────────┘          │
└──────────────────────────┘    └──────────────────────────────┘
                                        │
                                        ▼
┌─────────────────────────────────────────────────────────────────────┐
│              LOOP BACK TO QUERY ENGINE                               │
│  - Add tool result to conversation                                  │
│  - Continue AI processing                                           │
│  - Repeat until complete                                            │
└─────────────────────────────────────────────────────────────────────┘
```

## Key Design Principles

### 1. Streaming Everything

Claude Code streams virtually everything:
- AI text responses (character by character feel)
- Tool execution progress (you see what's happening)
- Permission prompts (don't wait to see what's being requested)
- Error messages (immediate feedback)

This creates the feeling of watching Claude "think" in real-time.

### 2. Permission-First

Every potentially dangerous operation requires explicit permission:
- File writes are blocked until approved
- Shell commands require confirmation
- Destructive commands (rm -rf) get extra warnings
- Permissions can be remembered for session or forever

### 3. Context is Everything

Claude Code doesn't just answer questions — it understands your project:
- Reads `.claude/` memory files
- Knows your git branch and status
- Understands your project structure
- Maintains conversation history
- Can access MCP servers for additional context

### 4. Graceful Degradation

The system handles failures at every level:
- API errors trigger automatic retries with backoff
- Permission denials are tracked and can escalate
- Network failures fall back to cached state
- Tool failures provide helpful error messages

### 5. Observable Operation

You can always see what's happening:
- Streaming output shows Claude's thinking
- Tool calls are displayed with their progress
- Permission prompts explain what's being requested
- Error messages suggest fixes

## The Development Experience

Using Claude Code feels like having a knowledgeable colleague who:

1. **Understands your project** — Sees all your files and git history
2. **Can run any command** — Has access to your terminal
3. **Thinks step by step** — You can watch the reasoning process
4. **Asks before acting** — Doesn't make unilateral decisions on risky ops
5. **Remembers context** — Knows what you've discussed in this session
6. **Can spawn helpers** — Can create sub-agents for parallel work

## Next Steps

- Read [Architecture Diagram](./architecture_diagram.md) for a visual representation
- Dive into [Modules Breakdown](../02_core_architecture/modules_breakdown.md) for detailed analysis
- Check [Execution Flow](../03_execution_flow/) for step-by-step tracing
