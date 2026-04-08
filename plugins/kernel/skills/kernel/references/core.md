# Core Types Reference

Foundational type vocabulary for the TAU kernel.

## Protocol Constants

```go
import "github.com/tailored-agentic-units/kernel/core/protocol"

protocol.Chat        // Text conversation
protocol.Vision      // Image understanding
protocol.Tools       // Function calling
protocol.Embeddings  // Vector embeddings
protocol.Audio       // Speech-to-text
```

### Message Type

```go
msg := protocol.Message{
    Role:    "user",
    Content: "Hello, world!",
}
```

## Response Types

```go
import "github.com/tailored-agentic-units/kernel/core/response"
```

| Type | Protocol | Primary Access |
|------|----------|----------------|
| `ChatResponse` | Chat, Vision | `resp.Choices[0].Message.Content` |
| `ToolsResponse` | Tools | `resp.Choices[0].Message.ToolCalls` |
| `EmbedResponse` | Embeddings | `resp.Data[0].Embedding` ([]float64) |
| `StreamingChunk` | ChatStream | `chunk.Content()` (string) |
| `AudioResponse` | Audio | `resp.Text` |

### Streaming

```go
for chunk := range chunks {
    if chunk.Error != nil {
        break
    }
    fmt.Print(chunk.Content())
}
```

## Configuration

```go
import "github.com/tailored-agentic-units/kernel/core/config"
```

### Loading Config

```go
cfg, err := config.LoadAgentConfig("config.json")
```

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

### Config Types

| Type | Purpose |
|------|---------|
| `AgentConfig` | Top-level agent configuration |
| `ProviderConfig` | Provider name, base URL, options |
| `ModelConfig` | Model name, capabilities per protocol |
| `ClientConfig` | HTTP timeout, retry policy |
| `Duration` | JSON-serializable duration with human-readable format |
