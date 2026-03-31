# API Layer: How Claude Code Talks to Anthropic

The API layer is the bridge between Claude Code's internal systems and Anthropic's Claude AI models. This document explains how that communication works.

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────┐
│                    Claude Code                              │
│  ┌─────────────────────────────────────────────────────┐   │
│  │              API Abstraction Layer                  │   │
│  │  ┌─────────────┐  ┌─────────────┐  ┌───────────┐  │   │
│  │  │ ClaudeAPI   │  │ RateLimits  │  │ CostTrack │  │   │
│  │  │ Client      │  │ Manager     │  │           │  │   │
│  │  └─────────────┘  └─────────────┘  └───────────┘  │   │
│  └─────────────────────────────────────────────────────┘   │
│                           │                                │
│  ┌─────────────────────────▼───────────────────────────┐   │
│  │              SDK Layer                               │   │
│  │         @anthropic-ai/sdk                           │   │
│  └─────────────────────────────────────────────────────┘   │
└───────────────────────────────┬─────────────────────────────┘
                                │ HTTPS
                                ▼
┌───────────────────────────────▼─────────────────────────────┐
│                    Anthropic API                            │
│    ┌──────────────┐  ┌──────────────┐  ┌──────────────┐   │
│    │ Messages API │  │  Beta APIs   │  │  Auth        │   │
│    └──────────────┘  └──────────────┘  └──────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

## Core API Client (services/api/claude.ts)

### Message Creation

The primary API call is `messages.create()`:

```typescript
// services/api/claude.ts
export async function createMessage(
  params: MessageParams
): Promise<Message> {
  // 1. Validate inputs
  validateParams(params)
  
  // 2. Normalize messages
  const normalizedMessages = normalizeMessagesForAPI(params.messages)
  
  // 3. Convert tools to API schema
  const apiTools = params.tools?.map(toolToAPISchema)
  
  // 4. Build request
  const request: MessageCreateParams = {
    model: params.model,
    max_tokens: params.maxTokens ?? getDefaultMaxTokens(params.model),
    messages: normalizedMessages,
    system: params.systemPrompt,
    tools: apiTools,
    thinking: params.thinkingConfig,
  }
  
  // 5. Execute with retries
  return withRetry(() => client.messages.create(request))
}
```

### Streaming Messages

```typescript
export async function streamMessage(
  params: MessageParams
): Promise<AsyncIterable<StreamEvent>> {
  const stream = await client.messages.stream({
    model: params.model,
    max_tokens: params.maxTokens ?? 8192,
    messages: params.messages,
    tools: params.tools?.map(toolToAPISchema),
    thinking: params.thinkingConfig,
  })
  
  return stream
}
```

### Stream Event Types

The API sends various event types during streaming:

```typescript
// Stream event types from the API
type StreamEvent =
  | { type: 'message_start', message: Message }
  | { type: 'content_block_start', index: number, content_block: ContentBlock }
  | { type: 'content_block_delta', index: number, delta: Delta }
  | { type: 'content_block_stop', index: number }
  | { type: 'message_delta', delta: MessageDelta, usage: Usage }
  | { type: 'message_stop' }

// Delta types
type Delta =
  | { type: 'text_delta', text: string }
  | { type: 'input_json_delta', text: string }
```

## Message Building (utils/messages.ts)

### Message Types

Claude Code uses a rich internal message format:

```typescript
type Message =
  | UserMessage
  | AssistantMessage
  | SystemMessage
  | ToolResultMessage
  | ProgressMessage
  | SystemLocalCommandMessage
  | HookResultMessage

interface UserMessage {
  type: 'user'
  content: string | ContentBlock[]
  timestamp: Date
}

interface AssistantMessage {
  type: 'assistant'
  content: string | ContentBlock[]
  toolCalls?: ToolCall[]
  usage?: Usage
  stopReason?: string
}

interface ToolResultMessage {
  type: 'tool_result'
  toolUseId: string
  content: string
  isError?: boolean
}
```

### Message Normalization

```typescript
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
        // System messages handled separately
        return null
    }
  }).filter(Boolean)
}
```

## Tool Schema Generation (utils/api.ts)

### Tool to API Schema Conversion

```typescript
export function toolToAPISchema(tool: Tool): ToolParam {
  return {
    name: tool.name,
    description: tool.description,
    input_schema: tool.inputSchema,
  }
}

// Example output:
{
  "name": "BashTool",
  "description": "Execute a bash command",
  "input_schema": {
    "type": "object",
    "properties": {
      "command": {
        "type": "string",
        "description": "The command to execute"
      },
      "timeout": {
        "type": "number",
        "description": "Timeout in milliseconds"
      }
    },
    "required": ["command"]
  }
}
```

## Request Configuration

### Model Selection

```typescript
// utils/model/model.ts
export function getModelConfig(model: string): ModelConfig {
  const configs: Record<string, ModelConfig> = {
    'claude-opus-4-5': {
      maxTokens: 8192,
      supportsVision: true,
      supportsTools: true,
      supportsThinking: true,
      costPer1kInput: 0.015,
      costPer1kOutput: 0.075,
    },
    'claude-sonnet-4-5': {
      maxTokens: 8192,
      supportsVision: true,
      supportsTools: true,
      supportsThinking: true,
      costPer1kInput: 0.003,
      costPer1kOutput: 0.015,
    },
    // ...
  }
  
  return configs[model] ?? getDefaultConfig()
}
```

### Thinking Configuration

```typescript
// When thinking is enabled:
const thinkingConfig = {
  type: 'enabled',
  budget_tokens: 10000,  // Max tokens for thinking
}

// This adds a "thinking" block before the final response
// Thinking tokens are billed at lower rate
```

## Response Handling (QueryEngine.ts)

### Event Processing

```typescript
export class QueryEngine {
  private accumulatedContent: Map<number, ContentBlock> = new Map()
  
  async *stream(): AsyncGenerator<StreamEvent> {
    for await (const event of this.apiStream) {
      switch (event.type) {
        case 'content_block_start':
          // Initialize new content block
          this.accumulatedContent.set(
            event.index,
            { type: event.content_block.type }
          )
          break
          
        case 'content_block_delta':
          const block = this.accumulatedContent.get(event.index)
          if (block.type === 'text') {
            block.text += event.delta.text
          } else if (block.type === 'tool_use') {
            if (!block.input) block.input = ''
            block.input += event.delta.input
          }
          break
          
        case 'content_block_stop':
          yield this.processCompletedBlock(
            this.accumulatedContent.get(event.index)
          )
          break
      }
    }
  }
}
```

### Stop Reasons

```typescript
// When streaming completes, the API provides a stop reason
type StopReason =
  | 'end_turn'           // Natural end of assistant turn
  | 'max_tokens'         // Hit token limit
  | 'stop_sequence'      // Found stop sequence
  | 'tool_use'           // Stopped to use a tool

// Handling different stop reasons:
function handleStopReason(reason: StopReason, usage: Usage) {
  switch (reason) {
    case 'end_turn':
      return { success: true, done: true }
    case 'max_tokens':
      return { success: true, done: false, warning: 'Token limit reached' }
    case 'tool_use':
      return { success: true, done: false }  // Normal, continue
    case 'stop_sequence':
      return { success: true, done: true }
  }
}
```

## Error Handling

### Error Types

```typescript
// services/api/errors.ts
export class APIError extends Error {
  constructor(
    message: string,
    public statusCode: number,
    public type: string,
    public isRetryable: boolean
  ) {
    super(message)
  }
}

// Common error types:
const ERROR_TYPES = {
  400: 'invalid_request_error',
  401: 'authentication_error',
  403: 'permission_error',
  429: 'rate_limit_error',
  500: 'api_error',
  529: 'overloaded_error',
}
```

### Retry Logic

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
      if (!isRetryableError(error) || attempt === maxRetries) {
        throw error
      }
      
      // Exponential backoff
      const delay = baseDelay * Math.pow(2, attempt)
      await sleep(delay)
    }
  }
  
  throw new Error('Should not reach here')
}

function isRetryableError(error: APIError): boolean {
  // Retry on 429 (rate limit), 500, 502, 503, 529
  return error.isRetryable
}
```

## Rate Limiting

### Rate Limit Manager

```typescript
// services/rateLimitMessages.ts
export class RateLimitManager {
  private limits: Map<string, RateLimitState> = new Map()
  
  async waitForRateLimit(model: string): Promise<void> {
    const limit = this.limits.get(model)
    if (!limit) return
    
    const now = Date.now()
    if (now < limit.resetAt) {
      const waitTime = limit.resetAt - now
      await sleep(waitTime)
    }
  }
  
  recordResponse(model: string, headers: ResponseHeaders) {
    const limit = parseRateLimitHeaders(headers)
    this.limits.set(model, {
      remaining: limit.remaining,
      resetAt: limit.reset,
    })
  }
}
```

## Cost Tracking (cost-tracker.ts)

### Token and Cost Accounting

```typescript
// cost-tracker.ts
export interface CostSummary {
  inputTokens: number
  outputTokens: number
  totalTokens: number
  inputCost: number
  outputCost: number
  totalCost: number
}

export class CostTracker {
  private sessionUsage: Usage[] = []
  
  recordUsage(usage: Usage, model: string) {
    this.sessionUsage.push(usage)
  }
  
  getSessionCost(): CostSummary {
    const summary = this.sessionUsage.reduce(
      (acc, usage) => ({
        inputTokens: acc.inputTokens + usage.input_tokens,
        outputTokens: acc.outputTokens + usage.output_tokens,
      }),
      { inputTokens: 0, outputTokens: 0 }
    )
    
    return {
      ...summary,
      totalTokens: summary.inputTokens + summary.outputTokens,
      inputCost: calculateCost(summary.inputTokens, 'input', 'opus'),
      outputCost: calculateCost(summary.outputTokens, 'output', 'opus'),
      totalCost: 0,  // Calculate from above
    }
  }
}
```

## Caching

### Response Caching

```typescript
// utils/fingerprint.ts
export function computeFingerprint(
  messages: Message[],
  model: string
): string {
  // Create a hash of the conversation state
  const content = JSON.stringify({
    messages: messages.map(m => ({
      type: m.type,
      content: m.content,
    })),
    model,
  })
  
  return createHash(content)
}

// Check cache before API call
async function queryWithCache(params: QueryParams): Promise<Response> {
  const fingerprint = computeFingerprint(params.messages, params.model)
  
  if (cache.has(fingerprint)) {
    return cache.get(fingerprint)
  }
  
  const response = await api.query(params)
  cache.set(fingerprint, response)
  
  return response
}
```

## Summary

The API layer handles:

1. **Message Building** — Rich internal format → API format
2. **Tool Schema Generation** — Claude Code tools → API tool definitions
3. **Streaming** — Real-time event processing
4. **Error Handling** — Retry logic and graceful failures
5. **Rate Limiting** — Respect API limits
6. **Cost Tracking** — Token and dollar accounting
7. **Caching** — Avoid redundant API calls
