# Kernel Reference

Runtime architecture and ConnectRPC interface.

## Architecture

The kernel composes all subsystems into a closed-loop agentic processing system:

```
core (types) ────────────────────────────────────────┐
agent (LLM communication) ───────────────────────────┤
tools (tool execution) ──────────────────────────────┤
session (conversation management) ───────────────────┼─→ kernel
memory (persistent memory) ──────────────────────────┤
skills (progressive disclosure) ─────────────────────┤
mcp (MCP client) ────────────────────────────────────┤
orchestrate (multi-agent coordination) ──────────────┘
```

Dependencies flow in one direction: core → capabilities → composition.

## Runtime Boundary Principle

- The kernel is a **closed-loop I/O system** with zero extension awareness
- The **ConnectRPC interface** is the sole extensibility boundary
- External services connect **through** the interface — the kernel never reaches out
- The same kernel serves embedded, desktop, server, or cloud deployments

## Extension Ecosystem

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

Defined in `rpc/proto/tau/kernel/v1/kernel.proto`:

```protobuf
service KernelService {
  rpc CreateSession(CreateSessionRequest) returns (CreateSessionResponse);
  rpc Run(RunRequest) returns (stream RunResponse);
  rpc InjectContext(InjectContextRequest) returns (InjectContextResponse);
  rpc GetSession(GetSessionRequest) returns (GetSessionResponse);
}
```

### Generated Code

```
rpc/gen/tau/kernel/v1/              # protobuf types (package kernelv1)
rpc/gen/tau/kernel/v1/kernelv1connect/  # ConnectRPC handlers and clients
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

### Proto Workflow

```bash
cd rpc && buf lint       # Lint proto definitions
cd rpc && buf generate   # Generate Go code
go vet ./...             # Verify generated code compiles
```
