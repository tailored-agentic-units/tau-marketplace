# Kernel Development

Development type reference for the TAU kernel monorepo. Augments the task execution workflow with kernel-specific conventions for implementation, testing, and validation.

## Stack and Tools

- **Language**: Go (current installed version)
- **Validate**: `go vet ./...`
- **Test**: `go test ./...`
- **Coverage (all)**: `go test ./... -coverprofile=coverage.out`
- **Coverage (pkg)**: `go test ./<package>/... -coverprofile=c.out`
- **Dependencies**: `go mod tidy`
- **Proto lint**: `cd rpc && buf lint`
- **Proto generate**: `cd rpc && buf generate`

## Project Conventions

### Directory Structure

```
kernel/
├── core/               # Foundational types
├── agent/              # LLM communication
├── orchestrate/        # Multi-agent coordination
├── memory/             # Persistent memory (skeleton)
├── tools/              # Tool execution (skeleton)
├── session/            # Conversation management (skeleton)
├── skills/             # Progressive disclosure (skeleton)
├── mcp/                # MCP client (skeleton)
├── kernel/             # Runtime loop (skeleton)
├── rpc/                # ConnectRPC infrastructure
├── cmd/                # Entry points
├── tests/              # Kernel-wide integration tests only
├── scripts/            # Infrastructure scripts
└── go.mod              # Single module
```

### Key Differences from go-library

- **No `pkg/` prefix**: Packages live directly under their subsystem root
- **Tests co-located**: `*_test.go` files alongside source (not separate `tests/` tree)
- **Top-level `tests/`**: Reserved for kernel-wide integration tests only
- **Single go.mod**: One module, one version, no dependency cascade
- **Single CHANGELOG.md**: Organized by subsystem sections (core, agent, orchestrate, kernel)
- **Single version tag**: One tag per release (no per-module tagging, no cascade)

### Naming

- Package names: short, singular, lowercase
- Files: lowercase, underscore-separated if needed
- Exported types: descriptive, no stuttering (`config.Config` not `config.ConfigType`)

### Dependency Management

- Single go.mod at kernel root
- Maintain unidirectional package dependency hierarchy (low-level to high-level)
- Shared types belong in the lowest layer that needs them
- No cross-module dependency cascade

## Implementation Patterns

When creating the implementation guide (Phase 3), apply these conventions:

- **Encapsulation**: Provide methods for accessing meaningful values. Do not expose or require direct field access to inner state.
- **Interfaces at boundaries**: Layers communicate through interfaces, not concrete types.
- **Configuration lifecycle**: Load config, transform to domain types, discard config. Domain objects do not hold config references.
- **Parameter encapsulation**: More than two parameters use a struct.
- **Error handling**: Wrap errors with context (`fmt.Errorf("operation failed: %w", err)`). Define package-level errors in `errors.go`. Use `Err` prefix for exported error variables.
- **Modern idioms**: Use modern Go features (`sync.WaitGroup.Go`, `for range n`, `min()`/`max()`, `errors.Join`).

## Testing Strategy

During Phase 5 (Testing), the AI implements and runs all testing:

### Test Organization

- Tests co-located with source in each package
- Example: `core/config/agent_test.go` tests `core/config/agent.go`

Co-located `*_test.go` files are the Go-idiomatic standard — used by the Go standard library, Kubernetes, Docker, and virtually all major Go projects. `go test` and coverage tooling expect this layout. Separate `tests/` directories are explicitly not the pattern for unit tests. The `package_test` (external test package) suffix provides the meaningful separation — black-box testing of the public API — without physical directory separation.

### Black-Box Testing

All tests use the external test package:

```go
package config_test

import (
    "testing"
    "github.com/tailored-agentic-units/kernel/core/config"
)
```

Test only the exported API. Do not test internal implementation details.

### Table-Driven Tests

```go
tests := []struct {
    name     string
    input    string
    expected string
}{
    {name: "valid input", input: "test", expected: "result"},
}

for _, tt := range tests {
    t.Run(tt.name, func(t *testing.T) {
        // test logic
    })
}
```

### HTTP Mocking

For packages that make HTTP calls:

```go
server := httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
    json.NewEncoder(w).Encode(mockResponse)
}))
defer server.Close()
```

### Asynchronous Test Synchronization

Tests that verify asynchronous behavior (message delivery, handler invocation, goroutine coordination) must use channel-based synchronization — never `time.Sleep`.

**Pattern:** Handler signals on a channel when invoked; test waits on channel with `select`/`time.After` deadline guard.

```go
handled := make(chan struct{}, 1)
handler := func(msg Message) error {
    // ... handler logic ...
    handled <- struct{}{}
    return nil
}

select {
case <-handled:
    // success — assert results
case <-time.After(5 * time.Second):
    t.Fatal("timed out waiting for handler")
}
```

**Key principles:**
- Channel signals replace `time.Sleep` for positive assertions (waiting for something to happen)
- Safety-net timeouts (5s) are generous — they prevent hangs, not measure timing
- For negative assertions (verifying something does NOT happen), use a short timeout (100-200ms) on channel receive
- Buffered channels (`make(chan struct{}, 1)`) prevent goroutine leaks if the test exits early

### Coverage Expectations

Focus on critical path coverage:

- Public interfaces and their implementations
- Exported functions and methods
- Request/response parsing
- Configuration loading
- Protocol routing and error paths

Uncovered code is acceptable when it consists of defensive error handling for OS-level failures or edge cases requiring special conditions.

## Validation Checklist

The AI runs through this checklist during Phase 6 (Validation):

- [ ] `go vet ./...` passes
- [ ] All existing tests pass: `go test ./...`
- [ ] New tests added for new/modified exported API
- [ ] Tests are black-box (`package <name>_test`)
- [ ] Tests are table-driven where applicable
- [ ] Coverage covers critical paths
- [ ] `go mod tidy` produces no changes
- [ ] No stuttering in type names
- [ ] Error messages: lowercase, no punctuation, wrapped with context
- [ ] `cd rpc && buf lint` passes (if proto changes were made)

## Closeout Augmentations

During task execution Phase 8c (CHANGELOG Update), add entries for the completed work under the dev tag section. Organize entries under the appropriate subsystem section within the `## v<target>-dev.<objective>.<issue>` heading.

- Entry format: `- [Description of change] (#[issue-number])`

## Release Augmentations

During a release session, apply these Go-specific conventions:

- **Tag format**: Must use `v` prefix (e.g., `v0.1.0`). Go module tooling requires this.
- **Single tag**: One tag for the entire kernel (no per-module tagging, no cascade).
- **Validation**: Run `go vet ./...`, `go test ./...`, and `cd rpc && buf lint` before tagging.
- **Post-release**: After pushing the tag, optionally warm the proxy cache:

```bash
GOPROXY=proxy.golang.org go list -m github.com/tailored-agentic-units/kernel@<version>
```

## Relevant Skills

Load these skills for kernel development:

- **tau:go-patterns** - interface design, error handling, package structure, configuration lifecycle
- **tau:kernel** - kernel subsystem reference (core, agent, orchestrate, runtime)
