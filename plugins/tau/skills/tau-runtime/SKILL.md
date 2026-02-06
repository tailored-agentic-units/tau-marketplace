---
name: tau-runtime
description: >
  Use when working with the tau-runtime — the agent runtime that composes
  all TAU libraries into a closed-loop processing system with a ConnectRPC interface.
  Triggers: import tau-runtime, RuntimeService, ConnectRPC, proto, runtime boundary,
  CreateSession, Run, InjectContext, GetSession, extension ecosystem.
---

# tau-runtime Usage Guide

Agent runtime for the TAU ecosystem. tau-runtime composes nine libraries into a
closed-loop agentic processing system, exposed through a ConnectRPC interface.

## When This Skill Applies

- Understanding the runtime architecture and library composition
- Working with the ConnectRPC interface (proto definitions, server, streaming)
- Understanding the runtime boundary principle
- Configuring and running the runtime

## Architecture

### Library Composition

```
tau-core (types) ─────────────────────────────────────┐
tau-agent (LLM communication) ────────────────────────┤
tau-tools (tool execution) ───────────────────────────┤
tau-session (conversation management) ────────────────┼─→ tau-runtime
tau-memory (persistent memory) ───────────────────────┤
tau-skills (progressive disclosure) ──────────────────┤
tau-mcp (MCP client) ─────────────────────────────────┤
tau-orchestrate (multi-agent coordination) ────────────┘
```

Dependencies flow in one direction: core → capabilities → composition.

### Runtime Boundary Principle

- The runtime is a **closed-loop I/O system** with zero extension awareness
- The **ConnectRPC interface** is the sole extensibility boundary
- External services connect **through** the interface — the runtime never reaches out
- The same runtime serves embedded, desktop, server, or cloud deployments

### Extension Ecosystem

External services connect through the ConnectRPC interface:

| Extension | Purpose |
|-----------|---------|
| Persistence | Session state, memory files |
| IAM | Authentication, authorization |
| Container | Sandbox management |
| MCP Gateway | Proxies MCP servers |
| Observability | Metrics, tracing export |
| UI | Web interface, CLI |

## ConnectRPC Interface

### Proto Definition

The service is defined in `proto/tau/runtime/v1/runtime.proto`:

```protobuf
service RuntimeService {
  rpc CreateSession(CreateSessionRequest) returns (CreateSessionResponse);
  rpc Run(RunRequest) returns (stream RunResponse);
  rpc InjectContext(InjectContextRequest) returns (InjectContextResponse);
  rpc GetSession(GetSessionRequest) returns (GetSessionResponse);
}
```

### Generated Code

```
gen/tau/runtime/v1/              # protobuf types
gen/tau/runtime/v1/runtimev1connect/  # ConnectRPC handlers and clients
```

### Key Types

| Type | Purpose |
|------|---------|
| `CreateSessionRequest` | Bootstrap context for a new session |
| `RunRequest` | Session ID + prompt to execute |
| `RunResponse` | Streamed events (token, tool_call, status, error) |
| `InjectContextRequest` | Add context to an active session |
| `ContextEntry` | Key-value context with content type |
| `SessionStatus` | ACTIVE, RUNNING, COMPLETED, ERROR |

## Development

### Prerequisites

- Go 1.25+
- `buf` CLI for proto management
- `protoc-gen-go` and `protoc-gen-connect-go`

### Proto Workflow

```bash
buf lint              # Lint proto definitions
buf generate          # Generate Go code
go vet ./...          # Verify generated code compiles
```

### Local Development

All TAU libraries use a `go.work` workspace at `~/tau/` for cross-repo development:

```
go 1.25.2

use (
    ./tau-core
    ./tau-agent
    ./tau-orchestrate
    ./tau-memory
    ./tau-tools
    ./tau-session
    ./tau-skills
    ./tau-mcp
    ./tau-runtime
)
```

## Related Skills

- **tau:tau-agent** — Agent creation, protocol execution, provider setup
- **tau:tau-core** — Foundational types: protocols, responses, configuration
- **tau:tau-orchestrate** — Multi-agent coordination, workflows, state graphs
