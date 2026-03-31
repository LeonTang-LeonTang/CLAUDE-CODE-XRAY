# Agent Logic: How Claude Thinks and Decides

The agent logic is what transforms Claude Code from a simple API wrapper into a capable coding assistant. This document explains how the AI model is prompted, guided, and enabled to accomplish complex tasks.

## The Agent Philosophy

Claude Code's agent isn't just "the AI model" — it's a carefully engineered system combining:

1. **System prompts** that define capabilities and constraints
2. **Tool definitions** that extend what the model can do
3. **Context gathering** that provides relevant information
4. **Decision frameworks** that guide the model's choices
5. **Feedback loops** that help the model learn and improve

## System Prompt Architecture

### Prompt Layers

```
┌─────────────────────────────────────────────────────────────┐
│                    Base System Prompt                       │
│  "You are Claude Code, a helpful coding assistant..."       │
├─────────────────────────────────────────────────────────────┤
│                   Project Context                           │
│  Git status, project type, recent changes                    │
├─────────────────────────────────────────────────────────────┤
│                    Memory Files                             │
│  .claude/memory.md, patterns learned                        │
├─────────────────────────────────────────────────────────────┤
│                    Tool Instructions                         │
│  Available tools, how to use them, best practices          │
├─────────────────────────────────────────────────────────────┤
│                    Session State                             │
│  Current task, what we've done so far                       │
└─────────────────────────────────────────────────────────────┘
```

### Base Prompt (constants/system.ts)

```typescript
// constants/system.ts
export const BASE_SYSTEM_PROMPT = `
You are Claude Code, an AI coding assistant made by Anthropic.

Your capabilities:
- Read, write, and edit files in the workspace
- Execute shell commands
- Search the web and fetch web pages
- Run tests and understand their output
- Explain code and architecture

Guidelines:
- Always prefer making minimal, focused changes
- Ask for clarification when the task is ambiguous
- Explain what you're doing, especially for complex changes
- Suggest improvements, but respect user preferences
- Be mindful of security when executing commands

Safety:
- Never execute destructive commands without confirmation
- Validate file paths to prevent directory traversal
- Sanitize command arguments to prevent injection
- Respect .gitignore and project conventions
`.trim()
```

### Project Context Injection

```typescript
// context.ts - builds project context
export async function buildProjectContext(): Promise<string> {
  const [gitStatus, branch, userName] = await Promise.all([
    getGitStatus(),
    getCurrentBranch(),
    getGitUserName(),
  ])
  
  return `
# Current Project

## Git Status
${gitStatus || 'Not a git repository'}

## Current Branch
${branch}

## Git User
${userName || 'Unknown'}
`.trim()
}
```

## Tool Prompt Engineering

### Tool Instructions

Each tool has carefully crafted instructions that guide the model:

```typescript
// tools/BashTool/prompt.ts
export const BASH_TOOL_PROMPT = `
## BashTool

Execute shell commands in the terminal.

**When to use:**
- Installing dependencies (npm install, pip install)
- Running build scripts (npm run build, make)
- Running tests (pytest, npm test)
- Checking file contents (cat, ls)
- Git operations (git status, git diff)

**When NOT to use:**
- For file operations (use FileEditTool or FileWriteTool instead)
- For long-running server processes (consider Task tools)

**Best practices:**
- Always check what a command does before running it
- Use absolute paths when possible
- Be aware of working directory
- Set reasonable timeouts for long-running commands

**Safety:**
- Destructive commands (rm -rf, format) require extra caution
- Commands that modify system files need confirmation
`.trim()
```

### Tool Selection Guidance

```typescript
// The model is guided to choose tools wisely:
const TOOL_SELECTION_GUIDANCE = `
## Tool Selection Strategy

1. **Read first, write second**: Read files before modifying them
2. **Incremental changes**: Make small, testable changes
3. **Use the right tool**:
   - Read files → FileReadTool
   - Edit files → FileEditTool
   - Create files → FileWriteTool
   - Find files → GlobTool
   - Search content → GrepTool
   - Run commands → BashTool
4. **Batch operations**: Group related operations together
5. **Verify results**: Check that changes worked as expected
`.trim()
```

## The Decision Framework

### When to Act vs. Ask

```typescript
// Guidance embedded in system prompt
const AUTONOMY_GUIDANCE = `
## Decision Making

**Act autonomously when:**
- The change is small and reversible
- The task is clearly specified
- You understand the codebase well
- You can verify the result

**Ask for confirmation when:**
- The change is large or complex
- The task is ambiguous
- You need more context
- The change affects multiple files
- The command is destructive or system-level
`.trim()
```

### Error Recovery Strategy

```typescript
// How Claude is guided to handle errors
const ERROR_RECOVERY_GUIDANCE = `
## Error Handling

When something goes wrong:

1. **Read the error carefully**
   - What exactly failed?
   - What was the expected outcome?
   - What might have caused it?

2. **Try to understand the root cause**
   - Check file contents
   - Run diagnostic commands
   - Look at recent changes

3. **Propose a fix**
   - Suggest specific changes
   - Explain why it should work
   - Offer to try it

4. **Verify the fix**
   - Run tests
   - Check the result
   - Ask for confirmation

Don't repeatedly try the same thing hoping for different results.
`.trim()
```

## Context Window Management

### Memory and Summarization

```typescript
// When conversation gets long, Claude is guided to compact
const COMPACTION_GUIDANCE = `
## Context Management

As the conversation grows, we'll need to summarize older messages.

When summarizing:
- Keep the essential information
- Preserve important decisions made
- Note any patterns or preferences learned
- Maintain file locations and structure

This helps maintain context quality as we work.
`.trim()
```

### Selective Context

```typescript
// Only include relevant context
export function filterRelevantContext(
  fullContext: Context,
  currentTask: Task
): Context {
  return {
    // Always include:
    projectType: fullContext.projectType,
    language: fullContext.language,
    
    // Include if relevant to task:
    relevantFiles: currentTask.relatedFiles(fullContext),
    relevantTests: currentTask.relatedTests(fullContext),
    
    // Exclude:
    // - Unrelated file contents
    // - Old conversation that's been summarized
    // - Debug output
  }
}
```

## Tool Use Reasoning

### Chain-of-Thought for Tools

The model is encouraged to reason about tool use:

```typescript
// Embedded in the tool instructions
const TOOL_USE_REASONING = `
## Thinking About Tool Use

Before calling a tool, think:

1. **What do I need to accomplish?**
   Break down the task into steps.

2. **What information do I need?**
   Do I need to read files first? Run commands?

3. **Which tool is best?**
   Consider efficiency and correctness.

4. **What will the result tell me?**
   Plan how to use the output.

5. **How do I verify success?**
   What checks should I run?

Example:
Task: "Add user authentication to the app"

Thought: "I need to understand the current structure first.
Let me check the project setup, then look at existing auth
patterns, then implement the changes."

1. GlobTool: Find existing route files
2. FileReadTool: Check app structure
3. FileReadTool: Look at existing patterns
4. FileWriteTool: Add auth middleware
5. BashTool: Run tests to verify
`.trim()
```

## Multi-Agent Coordination

### Agent Tool

When Claude spawns sub-agents:

```typescript
// tools/AgentTool/AgentTool.ts
export class AgentTool {
  async execute(input: AgentInput): Promise<AgentResult> {
    // 1. Create a sub-agent with specific task
    const agent = createAgent({
      task: input.task,
      model: input.model ?? getDefaultModel(),
      tools: input.allowedTools ?? getDefaultTools(),
    })
    
    // 2. Spawn agent task
    const task = await agent.spawn({
      cwd: input.cwd ?? getCwd(),
      agentId: input.agentId,
    })
    
    // 3. Stream results back
    for await (const update of task.stream()) {
      yield update
    }
    
    // 4. Return final result
    return task.result
  }
}
```

### Task Delegation

```typescript
// How Claude decides what to delegate
const DELEGATION_GUIDANCE = `
## Delegation

Consider delegating when:
- A task is independent and can run in parallel
- Multiple similar tasks need to be done
- A specialized agent would be more effective

Don't delegate:
- Tasks that need context from this conversation
- Tasks that depend on each other's results
- Quick, simple tasks (overhead not worth it)

When delegating:
- Be clear about what to do
- Provide necessary context
- Set expectations for the result
`.trim()
```

## Session Continuity

### Remembering State

```typescript
// The agent maintains awareness of:
const SESSION_STATE = `
## Session State

Throughout this conversation, I maintain awareness of:
- Current directory and project
- Files we've read and modified
- Commands we've run and their results
- User preferences and patterns
- Task progress and what's left to do

I don't need to re-read files I've already seen.
I remember what we've tried and what worked.
`.trim()
```

### Learning from Feedback

```typescript
// How the model adapts based on user feedback
export function applyUserFeedback(
  messages: Message[],
  feedback: UserFeedback
): Message[] {
  if (feedback.type === 'negative') {
    // Add to conversation as guidance
    const guidance = `
User feedback: "${feedback.comment}"

This approach didn't work well. Let me reconsider.
    `.trim()
    
    messages.push(createUserMessage(guidance))
  }
  
  // The model will naturally adapt in next response
  return messages
}
```

## Thinking Budget

### Extended Thinking

```typescript
// For complex tasks, thinking can be extended
const THINKING_CONFIG = {
  type: 'enabled',
  budget_tokens: 10000,  // Up to 10k tokens for thinking
}

// The model will think through the problem before responding
// Thinking is shown to the user (if enabled) or hidden
```

### Thinking Display

```typescript
// User can choose to see thinking
if (settings.showThinking) {
  // Stream thinking blocks as they appear
  streamToUI({
    type: 'thinking',
    content: thinkingBlock.content,
  })
}
```

## Summary

The agent logic in Claude Code is a sophisticated system that:

1. **Provides rich context** — Git status, project info, memory files
2. **Guides tool selection** — When to use which tool
3. **Encourages good practices** — Minimal changes, verification
4. **Manages autonomy** — Knows when to act vs. ask
5. **Handles errors gracefully** — Clear recovery strategies
6. **Coordinates multi-agent** — Can spawn sub-agents when needed
7. **Adapts to feedback** — Learns from user responses
8. **Manages context** — Compacts long conversations

The combination of smart prompting, rich tooling, and thoughtful guidance creates an agent that feels intelligent and capable while remaining safe and controllable.
