---
name: kernel
description: >
  Use when working with the TAU kernel — the single Go module containing all
  subsystems (core types, agent, orchestrate, runtime, and skeleton packages).
  Covers protocol constants, response types, configuration, agent creation,
  protocol execution (Chat, Vision, Tools, Embeddings), provider setup (Ollama, Azure),
  mock testing, hub coordination, messaging, state graphs, workflow patterns,
  observability, ConnectRPC interface, and the runtime boundary.
  Triggers: import kernel, KernelService, ConnectRPC, protocol.Chat, protocol.Vision,
  protocol.Tools, protocol.Embeddings, agent.New, Chat, Vision, Tools, Embed, ChatStream,
  Ollama setup, Azure setup, mock package, MockAgent, provider setup, hub.New,
  messaging.NewRequest, state.NewGraph, workflows.ProcessChain, workflows.ProcessParallel,
  workflows.ProcessConditional, Observer, CheckpointStore, config.Default, hub coordination,
  workflow patterns, state graphs, agent coordination, multi-agent, CreateSession, Run,
  InjectContext, GetSession, extension ecosystem, core types, response types.
---

# TAU Kernel Usage Guide

The TAU kernel (`github.com/tailored-agentic-units/kernel`) is a single Go module
containing integrated subsystems for the agent runtime.

## When This Skill Applies

- Working with core types (protocols, responses, configuration, model)
- Creating agents and executing protocols (Chat, Vision, Tools, Embeddings)
- Setting up providers (Ollama, Azure)
- Testing code that uses agents (mock package)
- Creating hubs and coordinating agents
- Building state graphs and workflow patterns
- Working with the ConnectRPC interface

## Module

```
github.com/tailored-agentic-units/kernel
```

Single module. All packages share one version. No dependency cascade.

## Subsystems

| Package | Purpose |
|---------|---------|
| `core/config` | Configuration types with human-readable durations |
| `core/protocol` | Protocol constants and Message type |
| `core/response` | Response types, parsing, and streaming |
| `core/model` | Model runtime type |
| `agent` | Agent interface, creation, protocol execution |
| `agent/client` | HTTP client with retry logic |
| `agent/providers` | Ollama, Azure implementations |
| `agent/request` | Protocol-specific request construction |
| `agent/mock` | MockAgent, MockClient, MockProvider, helpers |
| `orchestrate/config` | Hub, state, workflow configuration |
| `orchestrate/hub` | Multi-hub agent coordination |
| `orchestrate/messaging` | Message structures and builders |
| `orchestrate/observability` | Observer pattern (NoOp, Slog, Multi) |
| `orchestrate/state` | State graphs, checkpoints |
| `orchestrate/workflows` | Chain, Parallel, Conditional patterns |
| `rpc/` | ConnectRPC proto + generated code |

## Quick Start

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

## References

For detailed usage by subsystem:

- [Core types reference](references/core.md) — protocols, responses, config, model
- [Agent reference](references/agent.md) — agent creation, protocols, providers, mocks
- [Orchestrate reference](references/orchestrate.md) — hubs, messaging, state, workflows
- [Kernel reference](references/kernel.md) — runtime architecture, ConnectRPC interface
