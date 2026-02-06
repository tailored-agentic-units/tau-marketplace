---
name: tau-agent
description: >
  Use when building applications with the tau-agent library. Covers agent creation,
  protocol execution (Chat, Vision, Tools, Embeddings), provider setup (Ollama, Azure),
  and testing with mocks.
  Triggers: import tau-agent, agent.New, Chat, Vision, Tools, Embed, ChatStream,
  Ollama setup, Azure setup, mock package, MockAgent, provider setup.
---

# tau-agent Usage Guide

LLM communication layer for the TAU ecosystem. tau-agent provides the Agent interface,
HTTP client with retry, protocol-specific request handling, provider implementations,
and test mocks.

## When This Skill Applies

- Creating agents and executing protocols (Chat, Vision, Tools, Embeddings)
- Setting up providers (Ollama, Azure)
- Testing code that uses agents (mock package)
- Understanding response access patterns

## Packages

| Package | Purpose |
|---------|---------|
| `pkg/agent` | Agent interface, creation, protocol execution |
| `pkg/client` | HTTP client with retry logic |
| `pkg/providers` | Provider interface, Ollama, Azure implementations |
| `pkg/request` | Protocol-specific request construction |
| `pkg/mock` | MockAgent, MockClient, MockProvider, helpers |

## Getting Started

### Installation

```bash
go get github.com/tailored-agentic-units/tau-agent
```

### Basic Usage

```go
import (
    "github.com/tailored-agentic-units/tau-agent/pkg/agent"
    "github.com/tailored-agentic-units/tau-core/pkg/config"
)

cfg, _ := config.LoadAgentConfig("config.json")
a, _ := agent.New(cfg)

response, _ := a.Chat(ctx, "Hello, world!")
fmt.Println(response.Choices[0].Message.Content)
```

## Protocol Usage

### Chat

```go
resp, _ := a.Chat(ctx, "Explain Go interfaces")
content := resp.Choices[0].Message.Content
```

### Chat Streaming

```go
chunks, _ := a.ChatStream(ctx, "Tell me a story")
for chunk := range chunks {
    if chunk.Error != nil {
        log.Fatal(chunk.Error)
    }
    fmt.Print(chunk.Content())
}
```

### Vision

```go
images := []string{"./photo.jpg", "https://example.com/image.png"}
resp, _ := a.Vision(ctx, "Describe this image", images)
```

### Tools

```go
tools := []agent.Tool{
    {
        Name: "get_weather",
        Description: "Get weather for a location",
        Parameters: map[string]any{
            "type": "object",
            "properties": map[string]any{
                "location": map[string]any{"type": "string"},
            },
        },
    },
}
resp, _ := a.Tools(ctx, "What's the weather in Dallas?", tools)
// Handle resp.Choices[0].Message.ToolCalls
```

### Embeddings

```go
resp, _ := a.Embed(ctx, "Text to embed")
vector := resp.Data[0].Embedding // []float64
```

## Response Access Patterns

| Protocol | Return Type | Primary Access |
|----------|-------------|----------------|
| `Chat` | `*ChatResponse` | `resp.Choices[0].Message.Content` (string) |
| `ChatStream` | `<-chan StreamChunk` | `chunk.Content()` returns `string`; check `chunk.Error` |
| `Vision` | `*ChatResponse` | `resp.Choices[0].Message.Content` (string) |
| `Tools` | `*ChatResponse` | `resp.Choices[0].Message.ToolCalls` ([]ToolCall) |
| `Embed` | `*EmbedResponse` | `resp.Data[0].Embedding` ([]float64) |

### Option Precedence

1. Model config provides baseline defaults
2. Runtime options override config values
3. Model name always added automatically

```go
// Config: {"temperature": 0.7}
// Runtime override:
a.Chat(ctx, "prompt", map[string]any{"temperature": 0.9})
// Result: temperature = 0.9
```

## Provider Setup

### Ollama

```bash
docker compose up -d  # Starts Ollama with llama3.2:3b
```

```json
{
  "provider": {
    "name": "ollama",
    "base_url": "http://localhost:11434"
  }
}
```

### Azure (API Key)

```json
{
  "provider": {
    "name": "azure",
    "base_url": "https://your-resource.openai.azure.com/openai",
    "options": {
      "deployment": "gpt-4o",
      "api_version": "2025-01-01-preview",
      "auth_type": "api_key"
    }
  }
}
```

Pass token at runtime: `-token $AZURE_API_KEY`

### Azure (Entra ID)

Same config with `"auth_type": "bearer"`, pass bearer token.

## Testing with Mocks

```go
import "github.com/tailored-agentic-units/tau-agent/pkg/mock"
```

### Simple Mock

```go
mockAgent := mock.NewSimpleChatAgent("test-id", "Expected response")
result, _ := mockAgent.Chat(ctx, "any prompt")
// result contains "Expected response"
```

### Helper Constructors

| Constructor | Use Case |
|-------------|----------|
| `NewSimpleChatAgent(id, response)` | Basic chat |
| `NewStreamingChatAgent(id, chunks)` | Streaming |
| `NewToolsAgent(id, toolCalls)` | Tool calling |
| `NewEmbeddingsAgent(id, vector)` | Embeddings |
| `NewFailingAgent(id, err)` | Error handling |

## Error Handling

### Connection Errors

```go
resp, err := a.Chat(ctx, "prompt")
if err != nil {
    // Common: provider not running, wrong base_url, network issues
    log.Printf("connection error: %v", err)
}
```

### Timeouts

Controlled by `client.timeout` config. Retry policy handles transient failures.

### Context Cancellation

```go
ctx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
defer cancel()

resp, err := a.Chat(ctx, "prompt")
if errors.Is(err, context.DeadlineExceeded) {
    log.Println("request timed out")
}
```

## Agent Lifecycle

Agents are lightweight structs — no explicit cleanup needed. The underlying HTTP
client is reused efficiently.

```go
a1, _ := agent.New(cfg1)
a2, _ := agent.New(cfg2)
// No cleanup required — just let them go out of scope
```

## Related Skills

- **tau:tau-core** — Foundational types: protocols, responses, configuration
- **tau:tau-orchestrate** — Multi-agent coordination, workflows, and state graphs
