# Agent Reference

LLM communication layer for the TAU kernel.

## Getting Started

```go
import (
    "github.com/tailored-agentic-units/kernel/agent"
    "github.com/tailored-agentic-units/kernel/core/config"
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
```

### Embeddings

```go
resp, _ := a.Embed(ctx, "Text to embed")
vector := resp.Data[0].Embedding // []float64
```

## Response Access Patterns

| Protocol | Return Type | Primary Access |
|----------|-------------|----------------|
| `Chat` | `*ChatResponse` | `resp.Choices[0].Message.Content` |
| `ChatStream` | `<-chan StreamChunk` | `chunk.Content()`; check `chunk.Error` |
| `Vision` | `*ChatResponse` | `resp.Choices[0].Message.Content` |
| `Tools` | `*ChatResponse` | `resp.Choices[0].Message.ToolCalls` |
| `Embed` | `*EmbedResponse` | `resp.Data[0].Embedding` |

### Option Precedence

1. Model config provides baseline defaults
2. Runtime options override config values
3. Model name always added automatically

```go
a.Chat(ctx, "prompt", map[string]any{"temperature": 0.9})
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

### Azure (Entra ID)

Same config with `"auth_type": "bearer"`, pass bearer token.

## Testing with Mocks

```go
import "github.com/tailored-agentic-units/kernel/agent/mock"
```

### Helper Constructors

| Constructor | Use Case |
|-------------|----------|
| `NewSimpleChatAgent(id, response)` | Basic chat |
| `NewStreamingChatAgent(id, chunks)` | Streaming |
| `NewToolsAgent(id, toolCalls)` | Tool calling |
| `NewEmbeddingsAgent(id, vector)` | Embeddings |
| `NewFailingAgent(id, err)` | Error handling |

```go
mockAgent := mock.NewSimpleChatAgent("test-id", "Expected response")
result, _ := mockAgent.Chat(ctx, "any prompt")
```

## Agent Lifecycle

Agents are lightweight structs — no explicit cleanup needed. The underlying HTTP
client is reused efficiently.
