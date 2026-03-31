<p align="center">
  <img src="https://raw.githubusercontent.com/LeonTang-LeonTang/CLAUDE-CODE-XRAY/main/claude-code-xray/diagrams/logo.svg" alt="Claude Code Xray Logo" width="450"/>
</p>

<p align="center">
  <img src="https://img.shields.io/badge/License-MIT-blue?style=for-the-badge" alt="License"/>
  <img src="https://img.shields.io/badge/TypeScript-007ACC?style=for-the-badge&logo=typescript&logoColor=white" alt="TypeScript"/>
  <img src="https://img.shields.io/badge/Claude-AI-purple?style=for-the-badge&logo=anthropic&logoColor=white" alt="Claude"/>
</p>

---

## Watch the Video Tutorial

<p align="center">

[![Video Tutorial: Claude Code Xray - I Reverse Engineered Claude Code Here's How It Actually Works](https://raw.githubusercontent.com/LeonTang-LeonTang/CLAUDE-CODE-XRAY/main/claude-code-xray/video-thumbnail.png)](https://www.youtube.com/watch?v=YOUR_VIDEO_ID)

**[▶ Watch on YouTube](https://www.youtube.com/watch?v=YOUR_VIDEO_ID)**

</p>

> *"I Reverse Engineered Claude Code. Here's How It Actually Works."* — An 8–12 minute guided walkthrough of the entire repository, covering the 5-layer architecture, tool execution pipeline, hidden mechanics, and how to rebuild it yourself.

---

## Table of Contents

- [Overview](#overview)
- [Key Insights](#key-insights)
- [Project Statistics](#project-statistics)
- [Quick Navigation](#quick-navigation)
- [Reading Guide](#reading-guide)
- [Section-by-Section Deep Dive](#section-by-section-deep-dive)
  - [01_overview — The Big Picture](#01_overview--the-big-picture)
  - [02_core_architecture — Inside the Machine](#02_core_architecture--inside-the-machine)
  - [03_execution_flow — Follow the Journey](#03_execution_flow--follow-the-journey)
  - [04_key_components — Deep Dives](#04_key_components--deep-dives)
  - [05_code_walkthrough — Annotated Source](#05_code_walkthrough--annotated-source)
  - [06_design_patterns — The Blueprint](#06_design_patterns--the-blueprint)
  - [07_hidden_mechanics — The Secret Sauce](#07_hidden_mechanics--the-secret-sauce)
  - [08_rebuild_guide — Build Your Own](#08_rebuild_guide--build-your-own)
- [Visual Diagrams](#visual-diagrams)
- [Contributing](#contributing)
- [License](#license)

---

## Overview

> *"I Reverse Engineered Claude Code. Here's How It Actually Works."*

When I first started using Claude Code, I was amazed at how naturally it understood my codebase, ran commands, edited files, and seemed to *reason* about what I was building. It felt like magic — and as an engineer, that made me curious. **How does it actually work?**

This documentation project is the result of **deep reverse-engineering analysis** of Claude Code's source code. It's written for developers who want to understand not just *what* Claude Code does, but *how* and *why* it works the way it does.

### What You'll Learn

<p align="center">

| Topic                          | Description                                                  |
| ------------------------------ | ------------------------------------------------------------ |
| **5-Layer Architecture** | CLI → Bridge → Query Engine → Tool System → State        |
| **Message Flow**         | User input → AI processing → tool execution → UI feedback |
| **Clever Tricks**        | Context compaction, speculative execution, MCP integration   |
| **Design Decisions**     | Multi-agent teams, remote sessions, worktree isolation       |
| **Build It Yourself**    | Rebuild key components from scratch                          |

</p>

---

## Key Insights

### 1. It's Not Just an API Wrapper

Claude Code is often assumed to be a simple wrapper around the Anthropic API. In reality, it's a sophisticated **agent orchestration system** with:

| Feature                                  | Description                                                                   |
| ---------------------------------------- | ----------------------------------------------------------------------------- |
| **Concurrent Execution**           | Background tasks, agent sub-processes, and shell commands all run in parallel |
| **Custom Streaming**               | Rich event system for progress, errors, and tool results                      |
| **Permission State Machines**      | Every potentially dangerous operation goes through a permission model         |
| **Intelligent Context Management** | Not just truncation, but intelligent compaction and summarization             |

### 2. The REPL is Actually a React App

Surprisingly, Claude Code's terminal interface (`ink/`) is built with React! It uses a custom React reconciler that renders to terminal ANSI escape codes instead of HTML DOM.

```
┌─────────────────────────────────────────────────────────────┐
│                    Architecture at a Glance                 │
├─────────────────────────────────────────────────────────────┤
│  ┌──────────────────────────────────────────────────────┐  │
│  │              CLI LAYER (main.tsx)                      │  │
│  │  Commands │ REPL │ Terminal │ Setup                   │  │
│  └──────────────────────────────────────────────────────┘  │
│                            ↓                                │
│  ┌──────────────────────────────────────────────────────┐  │
│  │              BRIDGE LAYER (bridge/)                   │  │
│  │  Remote │ Session │ Permission │ Transport            │  │
│  └──────────────────────────────────────────────────────┘  │
│                            ↓                                │
│  ┌──────────────────────────────────────────────────────┐  │
│  │         QUERY ENGINE (query.ts, QueryEngine.ts)       │  │
│  │  Message Pipeline │ Context Builder │ Tool Loop │ Stream│  │
│  └──────────────────────────────────────────────────────┘  │
│                            ↓                                │
│  ┌──────────────────────────────────────────────────────┐  │
│  │              TOOL SYSTEM (tools/)                     │  │
│  │  Bash │ File │ Web │ Agent │ MCP Tools                │  │
│  └──────────────────────────────────────────────────────┘  │
│                            ↓                                │
│  ┌──────────────────────────────────────────────────────┐  │
│  │              STATE LAYER (state/)                     │  │
│  │  AppState │ Store │ Hooks │ Permissions               │  │
│  └──────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
```

### 3. MCP is a First-Class Citizen

The Model Context Protocol isn't an afterthought — it's deeply integrated:

- MCP servers are managed through the MCP client service
- Tools are automatically discovered and wrapped
- Resources can be read via `ReadMcpResourceTool`
- Auth is handled through `McpAuthTool`

### 4. Dead Code Elimination is Used Extensively

Claude Code uses `feature()` flags from Bun to completely remove unused code at bundle time:

```typescript
const SleepTool = feature('PROACTIVE') || feature('KAIROS')
  ? require('./tools/SleepTool/SleepTool.js').SleepTool
  : null
```

### 5. The Tool System is a Pipeline

```
┌─────────┐    ┌──────────┐    ┌─────────┐    ┌──────────┐    ┌────────┐
│ Request │ -> │Permission│ -> │Speculative│ -> │Execution │ -> │Result  │
└─────────┘    └──────────┘    └─────────┘    └──────────┘    └────────┘
                                                     ↓
                                            ┌──────────────┐
                                            │Progress Stream│
                                            └──────────────┘
```

---

## Project Statistics

| Metric               | Value     | Visual |
| -------------------- | --------- | ------ |
| Source Files         | ~2,000+   | 📁     |
| TypeScript Files     | ~1,800    | 📝     |
| React Components     | ~150      | ⚛️   |
| Tool Implementations | ~46       | 🔧     |
| Command Handlers     | ~100+     | ⌨️   |
| Services             | ~40       | ⚙️   |
| Lines of Code        | ~800,000+ | 📊     |

---

## Quick Navigation

| Section                                      | What You'll Find                                      | Difficulty   |
| -------------------------------------------- | ----------------------------------------------------- | ------------ |
| [01_overview](./claude-code-xray/01_overview/)                   | High-level system overview and architecture diagrams  | Beginner     |
| [02_core_architecture](./claude-code-xray/02_core_architecture/) | Deep dive into each module and their dependencies     | Advanced     |
| [03_execution_flow](./claude-code-xray/03_execution_flow/)       | Step-by-step tracing of request to response           | Intermediate |
| [04_key_components](./claude-code-xray/04_key_components/)       | Detailed breakdown of API, Agent, Memory, Tool layers | Advanced     |
| [05_code_walkthrough](./claude-code-xray/05_code_walkthrough/)   | Annotated walkthrough of critical files               | Intermediate |
| [06_design_patterns](./claude-code-xray/06_design_patterns/)     | Design patterns and abstractions used                 | Advanced     |
| [07_hidden_mechanics](./claude-code-xray/07_hidden_mechanics/)   | Non-obvious behaviors and edge cases                  | Expert       |
| [08_rebuild_guide](./claude-code-xray/08_rebuild_guide/)         | How to rebuild key components                         | All Levels   |

---

## Reading Guide

| If You're...                               | Start Here                                    | Then Read                                                                         |
| ------------------------------------------ | --------------------------------------------- | --------------------------------------------------------------------------------- |
| **New to Claude Code?**              | `claude-code-xray/01_overview/system_overview.md`            | `claude-code-xray/03_execution_flow/request_to_response_flow.md`                                 |
| **Want to build something similar?** | `claude-code-xray/05_code_walkthrough/entry_point.md`        | `claude-code-xray/06_design_patterns/patterns_used.md` → `claude-code-xray/08_rebuild_guide/how_to_rebuild.md` |
| **Deep dive?**                       | `claude-code-xray/02_core_architecture/modules_breakdown.md` | `claude-code-xray/07_hidden_mechanics/non_obvious_behaviors.md`                                  |

---

## Section-by-Section Deep Dive

### 📊 01_overview — The Big Picture

This section is your **starting point**. It provides the high-level architecture view that every explorer needs.

**What's Inside:**

| File                        | Description                                                               |
| --------------------------- | ------------------------------------------------------------------------- |
| `system_overview.md`      | The 5-layer architecture: CLI, Bridge, Query Engine, Tool System, State   |
| `architecture_diagram.md` | Mermaid diagrams showing system architecture, message flow, tool pipeline |

> 💡 **Why Start Here?** Even if you're an expert, this section gives you the mental map you need before diving into specifics.

---

### 🏗️ 02_core_architecture — Inside the Machine

This is where we **dissect each layer** and understand how components interact.

**What's Inside:**

| File                     | Description                                                                                                                                                                                        |
| ------------------------ | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `modules_breakdown.md` | The most detailed document. Explains every major module: main.tsx (~4600 lines), bridgeMain.ts (~3000 lines), query.ts (~1700 lines), QueryEngine.ts (~1200 lines), Tool.ts (~700 lines), and more |
| `dependency_graph.md`  | Startup dependency order, query chain, tool execution chain, permission chain, import patterns                                                                                                     |

> 💡 **Why This Section?** This is the **meat** of the documentation. If you want to understand HOW something works in detail, it's here.

---

### ⚡ 03_execution_flow — Follow the Journey

This section traces **exactly what happens** when you type a message.

**What's Inside:**

| File                            | Description                                                                                                                                                     |
| ------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `request_to_response_flow.md` | 9-step trace: CLI Input → Process → Build Context → Query → API Call → Stream Response → Handle Tools → Continue Loop → Final Response                  |
| `lifecycle.md`                | Complete lifecycle: Bootstrap, Config Loading, Auth, MCP Connection, REPL Launch, Query Loop, Tool Execution, Compaction, Session Management, Graceful Shutdown |

> 💡 **Why This Section?** Perfect for understanding the **temporal aspect** — what happens when, and in what order.

---

### 🎯 04_key_components — Deep Dives

Four comprehensive guides to the most important subsystems.

**What's Inside:**

| File                 | Topic                                                                                                                                                        |
| -------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `api_layer.md`     | Message creation, streaming, tool schema generation, request config, response handling, error types, retry logic, rate limiting, cost tracking               |
| `agent_logic.md`   | System prompt architecture, tool prompt engineering, decision framework, error recovery, context window management, tool reasoning, multi-agent coordination |
| `memory_system.md` | Session memory, context compaction, persistent memory (.claude/), implicit memory, memory management commands, team memory sync                              |
| `tool_usage.md`    | Tool registry, orchestration pipeline, permission flow, BashTool, FileEditTool, MCPTool, progress streaming, security                                        |

> 💡 **Why This Section?** These four documents cover the **core systems** that make Claude Code actually useful.

---

### 📖 05_code_walkthrough — Annotated Source

Where we **read the code together** and explain what each critical file does.

**What's Inside:**

| File                       | Description                                                                                                                                                    |
| -------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `entry_point.md`         | Startup sequence: Profile checkpoints, prefetch optimization, feature flags, command registration, REPL launch                                                 |
| `critical_files.md`      | 15 most important files ranked: Tier 1 (Critical), Tier 2 (Important), Tier 3 (Significant)                                                                    |
| `important_functions.md` | Key algorithms:`query()`, `buildQueryContext()`, `streamMessage()`, `runTools()`, `checkCommandSecurity()`, `compactMessages()`, `createStore()` |

> 💡 **Why This Section?** For developers who want to **see actual code patterns** and understand implementation details.

---

### 🎨 06_design_patterns — The Blueprint

This section reveals the **architectural patterns** that make Claude Code maintainable.

**What's Inside:**

| #  | Pattern                          | Description                  |
| -- | -------------------------------- | ---------------------------- |
| 1  | Async Generator Pipeline         | Streaming throughout         |
| 2  | Simple Reactive Store            | 35 lines of state management |
| 3  | Dependency Injection via Context | Decoupling tools             |
| 4  | Feature Flags for DCE            | Lean builds                  |
| 5  | Lazy Module Loading              | Breaking circular deps       |
| 6  | Pipeline Pattern                 | Composable stages            |
| 7  | Strategy Pattern                 | Swappable algorithms         |
| 8  | Observer Pattern                 | React to state changes       |
| 9  | Factory Pattern                  | Creating similar objects     |
| 10 | Command Pattern                  | Encapsulating operations     |
| 11 | Memoization                      | Caching expensive calls      |
| 12 | Builder Pattern                  | Complex construction         |
| 13 | Retry with Backoff               | Handling transients          |
| 14 | Event Emitter                    | Notifying listeners          |
| 15 | Partitioning by Type             | Grouping for concurrency     |

> 💡 **Why This Section?** These patterns are **reusable** — you can apply them to your own projects!

---

### 🧩 07_hidden_mechanics — The Secret Sauce

This is where we reveal the **surprising, clever, non-obvious** things.

**What's Inside:**

| #  | Hidden Gem                  | Description                  |
| -- | --------------------------- | ---------------------------- |
| 1  | Prefetch Optimization       | I/O before imports!          |
| 2  | Dead Code Elimination       | Compile-time feature removal |
| 3  | Async Generators Throughout | Nothing buffers              |
| 4  | Concurrency Partitioning    | Read-only tools run parallel |
| 5  | Reverse Edit Application    | Correct multi-edit           |
| 6  | Memoized Git Status         | Cached expensive calls       |
| 7  | Compaction Boundaries       | Long conversation support    |
| 8  | Permission State Machine    | Enforced transitions         |
| 9  | React Without DOM           | Ink reconciler               |
| 10 | Token Estimation            | Fast vs. accurate            |
| 11 | Profile Checkpoints         | Built-in profiling           |
| 12 | Dynamic MCP Wrapping        | Runtime discovery            |
| 13 | Capacity Wake               | Efficient background tasks   |
| 14 | Session Fingerprinting      | Change detection             |

> 💡 **Why This Section?** These are the **"aha!" moments** — the things that make experienced developers say "oh, that's clever!"

---

### 🔨 08_rebuild_guide — Build Your Own

The ultimate hands-on section. **Build a Claude Code-style system from scratch.**

**12 Steps from Zero to Working Agent:**

| Step | Component                | What You Build              |
| ---- | ------------------------ | --------------------------- |
| 1    | Minimal Reactive Store   | State management            |
| 2    | Tool Interface           | The tool contract           |
| 3    | Tool Registry            | Centralized tool management |
| 4    | Tool Orchestration       | Pipeline execution          |
| 5    | Concurrency Partitioning | Safe parallelism            |
| 6    | Simple API Client        | Anthropic integration       |
| 7    | Streaming API Client     | Real-time responses         |
| 8    | Query Orchestration      | Combines everything         |
| 9    | Terminal UI              | User interaction            |
| 10   | Entry Point              | Putting it together         |
| 11   | Permission System        | Safety gates                |
| 12   | Context Building         | Rich system prompts         |

> 💡 **Why This Section?** Because **understanding becomes mastery** when you build it yourself.

---

## Visual Diagrams

Looking for visual representations? Check out the **`[diagrams](./claude-code-xray/diagrams/)`** folder:

### 📊 Diagrams — View Directly in Markdown

| Diagram | Description |
| ------- | ----------- |
| [architecture-overview.png](./claude-code-xray/diagrams/architecture-overview.png) | High-level 5-layer architecture |
| [tool-execution-pipeline.png](./claude-code-xray/diagrams/tool-execution-pipeline.png) | 4-step tool execution |
| [flow-overview.png](./claude-code-xray/diagrams/flow-overview.png) | 8-step request-response flow |
| [startup-sequence.png](./claude-code-xray/diagrams/startup-sequence.png) | Startup with prefetch optimization |

#### Architecture Overview

![Architecture Overview](https://raw.githubusercontent.com/LeonTang-LeonTang/CLAUDE-CODE-XRAY/main/claude-code-xray/diagrams/architecture-overview.png)

#### Tool Execution Pipeline

![Tool Execution Pipeline](https://raw.githubusercontent.com/LeonTang-LeonTang/CLAUDE-CODE-XRAY/main/claude-code-xray/diagrams/tool-execution-pipeline.png)

#### Flow Overview

![Flow Overview](https://raw.githubusercontent.com/LeonTang-LeonTang/CLAUDE-CODE-XRAY/main/claude-code-xray/diagrams/flow-overview.png)

#### Startup Sequence

![Startup Sequence](https://raw.githubusercontent.com/LeonTang-LeonTang/CLAUDE-CODE-XRAY/main/claude-code-xray/diagrams/startup-sequence.png)

---

### 🧠 Interactive Mind Maps — Full Browser Experience

Open these HTML files for the complete interactive experience:

| File                                                               | What's Inside                                              |
| ------------------------------------------------------------------ | ---------------------------------------------------------- |
| [`architecture-mindmap.html`](./claude-code-xray/diagrams/architecture-mindmap.html) | 5-layer architecture, tool pipeline, request-response flow |
| [`flow-mindmap.html`](./claude-code-xray/diagrams/flow-mindmap.html)                 | Complete flow, startup sequence, session lifecycle         |

```bash
# Open in browser
open claude-code-xray/diagrams/architecture-mindmap.html
open claude-code-xray/diagrams/flow-mindmap.html
```

---

### 📐 Mermaid Diagrams

| File                 | Description                           |
| -------------------- | ------------------------------------- |
| `architecture.mmd` | System architecture in Mermaid format |
| `flow.mmd`         | Execution flow in Mermaid format      |

---

## Who This Is For

| Audience                      | Why It Matters                                          |
| ----------------------------- | ------------------------------------------------------- |
| 👨‍💻 Curious Developers     | Understand how AI coding assistants work under the hood |
| 🤖 AI Engineers               | Build your own agentic systems                          |
| 🛠️ Open Source Contributors | Understand Claude Code's architecture                   |
| ⚡ Power Users                | Know the tool's capabilities and limitations            |
| 📚 Educators                  | Teach about LLM-based agent systems                     |

---

## Contributing

This is an independent reverse-engineering project and is not affiliated with Anthropic. It's meant for educational purposes and to satisfy engineering curiosity.

If you find errors or want to add insights, feel free to contribute!

---

## License

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](https://opensource.org/licenses/MIT)

MIT License — Use this knowledge to build amazing things.

---

<p align="center">

*"The best way to understand a system is to take it apart and put it back together."*

— Adapted from Grace Hopper

</p>
# CLAUDE-CODE-XRAY
