---
name: tau-core
description: >
  Use when working with the tau-core library — the foundational type vocabulary
  for the TAU ecosystem. Covers protocol constants, response types, configuration
  structures, and model types.
  Triggers: import tau-core, protocol.Chat, protocol.Vision, protocol.Tools,
  protocol.Embeddings, ChatResponse, ToolsResponse, config.LoadAgentConfig,
  AgentConfig, ProviderConfig, ModelConfig, ClientConfig, Duration.
---

# tau-core Usage Guide

Foundational type vocabulary for the TAU ecosystem. tau-core defines the shared
types that all other TAU libraries build upon.

> **Note**: For agent creation, protocol execution, provider setup, and mock testing,
> see the **tau:tau-agent** skill. tau-agent was extracted from tau-core and contains
> the LLM communication layer.

## When This Skill Applies

- Working with protocol constants (Chat, Vision, Tools, Embeddings, Audio)
- Using response types (ChatResponse, ToolsResponse, EmbedResponse, etc.)
- Loading or constructing configuration (AgentConfig, ProviderConfig, ModelConfig)
- Working with the Model runtime type

## Packages

| Package | Purpose |
|---------|---------|
| `pkg/protocol` | Protocol constants and Message type |
| `pkg/response` | Response types, parsing, and streaming support |
| `pkg/config` | Configuration types with human-readable durations |
| `pkg/model` | Model runtime type bridging config to execution |

## Protocol Constants

```go
import "github.com/tailored-agentic-units/tau-core/pkg/protocol"

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
import "github.com/tailored-agentic-units/tau-core/pkg/response"
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
// StreamingChunk provides Content() for easy text extraction
for chunk := range chunks {
    if chunk.Error != nil {
        break
    }
    fmt.Print(chunk.Content())
}
```

## Configuration

```go
import "github.com/tailored-agentic-units/tau-core/pkg/config"
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

## Related Skills

- **tau:tau-agent** — Agent creation, protocol execution, provider setup, mock testing
- **tau:tau-orchestrate** — Multi-agent coordination, workflows, and state graphs
