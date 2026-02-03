---
name: tau-core
description: >
  Use when building applications with the tau-core library. Covers installation,
  configuration, protocol usage (Chat, Vision, Tools, Embeddings),
  provider setup, and testing with mocks.
  Triggers: import tau-core, agent.New, Chat, Vision, Tools, Embed,
  config.json, Ollama setup, Azure setup, mock package.
---

# tau-core Usage Guide

## When This Skill Applies

- Installing and configuring tau-core
- Creating agents and executing protocols
- Setting up providers (Ollama, Azure)
- Testing code that uses tau-core
- Understanding response access patterns

## Getting Started

### Installation

```bash
go get github.com/tailored-agentic-units/tau-core
```

### Basic Usage

```go
import (
    "github.com/tailored-agentic-units/tau-core/pkg/agent"
    "github.com/tailored-agentic-units/tau-core/pkg/config"
)

cfg, _ := config.LoadAgentConfig("config.json")
a, _ := agent.New(cfg)

response, _ := a.Chat(ctx, "Hello, world!")
fmt.Println(response.Choices[0].Message.Content)
```

## Configuration

### Config Structure

```json
{
  "name": "my-agent",
  "system_prompt": "You are a helpful assistant",
  "client": {
    "timeout": "24s",
    "retry": {
      "max_retries": 3,
      "initial_backoff": "1s"
    }
  },
  "provider": {
    "name": "ollama",
    "base_url": "http://localhost:11434"
  },
  "model": {
    "name": "llama3.2:3b",
    "capabilities": {
      "chat": {"max_tokens": 4096, "temperature": 0.7}
    }
  }
}
```

### Duration Format

Supports human-readable strings: `"24s"`, `"1m"`, `"2h"`

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
// Local file or URL
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

Each protocol returns a typed response. Here are the common access patterns:

| Protocol | Return Type | Primary Access |
|----------|-------------|----------------|
| `Chat` | `*ChatResponse` | `resp.Choices[0].Message.Content` (string) |
| `ChatStream` | `<-chan StreamChunk` | `chunk.Content()` returns `string`; check `chunk.Error` |
| `Vision` | `*ChatResponse` | `resp.Choices[0].Message.Content` (string) |
| `Tools` | `*ChatResponse` | `resp.Choices[0].Message.ToolCalls` ([]ToolCall) |
| `Embed` | `*EmbedResponse` | `resp.Data[0].Embedding` ([]float64) |

### Streaming Helper

The `Content()` method on stream chunks extracts text content directly:

```go
chunks, _ := a.ChatStream(ctx, "prompt")
for chunk := range chunks {
    if chunk.Error != nil {
        // Handle error
        break
    }
    text := chunk.Content() // string - ready to use
    fmt.Print(text)
}
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

### Simple Mock

```go
import "github.com/tailored-agentic-units/tau-core/pkg/mock"

mockAgent := mock.NewSimpleChatAgent("test-id", "Expected response")

// Use mockAgent in your code
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

Connection errors occur when the provider endpoint is unreachable or the network fails.

```go
resp, err := a.Chat(ctx, "prompt")
if err != nil {
    // Check for connection-level errors
    // Common causes: provider not running, wrong base_url, network issues
    log.Printf("connection error: %v", err)
    return err
}
```

**Resolution**: Verify the provider is running (`docker ps` for Ollama), check `base_url` in config, and ensure network connectivity.

### Model Not Found

Occurs when the configured model is not available on the provider.

```go
// Error: "model 'llama3.2:3b' not found"
// Resolution: Pull the model first
// For Ollama: ollama pull llama3.2:3b
```

**Resolution**: Verify the model name matches exactly what the provider expects. For Ollama, use `ollama list` to see available models. For Azure, confirm the deployment name in the portal.

### Timeouts

Controlled by the `client.timeout` config field. The retry policy handles transient failures automatically.

```go
// Config: {"client": {"timeout": "24s", "retry": {"max_retries": 3, "initial_backoff": "1s"}}}
// After max_retries exhausted, the final error is returned
resp, err := a.Chat(ctx, "long prompt")
if err != nil {
    // May be a timeout after all retries exhausted
    log.Printf("request failed after retries: %v", err)
}
```

**Resolution**: Increase `client.timeout` for long-running requests. Adjust `retry.max_retries` and `retry.initial_backoff` for transient failures.

### Context Cancellation

All protocol methods respect context cancellation:

```go
ctx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
defer cancel()

resp, err := a.Chat(ctx, "prompt")
if errors.Is(err, context.DeadlineExceeded) {
    log.Println("request timed out")
}
```

## Agent Lifecycle

Agents are lightweight structs that hold configuration and a client reference. There is no explicit cleanup or shutdown needed -- agents can be created and discarded freely. The underlying HTTP client is reused efficiently.

```go
// Create agents as needed -- they are inexpensive
a1, _ := agent.New(cfg1)
a2, _ := agent.New(cfg2)

// No cleanup required -- just let them go out of scope
// The garbage collector handles the rest
```

When using agents within **tau:tau-orchestrate** hubs, the hub manages message routing but does not own agent lifecycle. Agents registered with a hub remain valid after hub shutdown.

## Related Skills

- **tau:tau-orchestrate** - Multi-agent coordination, workflows, and state graphs built on top of tau-core
