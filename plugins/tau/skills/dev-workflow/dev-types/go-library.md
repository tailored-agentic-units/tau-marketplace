# Go Library Development

Development type reference for Go library projects. Augments the task execution workflow with Go-specific conventions for implementation, testing, and validation.

## Stack and Tools

- **Language**: Go (current installed version)
- **Validate**: `go vet ./...`
- **Test**: `go test ./tests/...`
- **Coverage**: `go test ./tests/... -coverprofile=coverage.out -coverpkg=./pkg/...`
- **Dependencies**: `go mod tidy`

## Project Conventions

### Directory Structure

```
project/
├── pkg/              # Public library packages
├── cmd/              # CLI utilities (if any)
├── tests/            # Black-box tests (mirrors pkg/ structure)
└── go.mod
```

### Naming

- Package names: short, singular, lowercase
- Files: lowercase, underscore-separated if needed
- Exported types: descriptive, no stuttering (`config.Config` not `config.ConfigType`)

### Dependency Management

- Minimize external dependencies
- Maintain unidirectional package dependency hierarchy (low-level to high-level)
- Shared types belong in the lowest layer that needs them

## Implementation Patterns

When creating the implementation guide (Phase 3), apply these conventions:

- **Encapsulation**: Provide methods for accessing meaningful values. Do not expose or require direct field access to inner state.
- **Interfaces at boundaries**: Layers communicate through interfaces, not concrete types.
- **Configuration lifecycle**: Load config, transform to domain types, discard config. Domain objects do not hold config references.
- **Parameter encapsulation**: More than two parameters use a struct.
- **Error handling**: Wrap errors with context (`fmt.Errorf("operation failed: %w", err)`). Define package-level errors in `errors.go`. Use `Err` prefix for exported error variables.
- **Modern idioms**: Use modern Go features (`sync.WaitGroup.Go`, `for range n`, `min()`/`max()`, `errors.Join`).

## Testing Strategy

During Phase 5 (Validation), the AI implements and runs all testing:

### Test Organization

- Tests live in `tests/` directory mirroring `pkg/` structure
- Example: `tests/config/` tests `pkg/config/`

### Black-Box Testing

All tests use the external test package:

```go
package config_test

import (
    "testing"
    "github.com/owner/project/pkg/config"
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

### Coverage Expectations

Focus on critical path coverage:

- Public interfaces and their implementations
- Exported functions and methods
- Request/response parsing
- Configuration loading
- Protocol routing and error paths

Uncovered code is acceptable when it consists of defensive error handling for OS-level failures or edge cases requiring special conditions.

## Validation Checklist

The AI runs through this checklist during Phase 5:

- [ ] `go vet ./...` passes
- [ ] All existing tests pass: `go test ./tests/...`
- [ ] New tests added for new/modified exported API
- [ ] Tests are black-box (`package <name>_test`)
- [ ] Tests are table-driven where applicable
- [ ] Coverage covers critical paths: `go test ./tests/... -coverprofile=coverage.out -coverpkg=./pkg/...`
- [ ] `go mod tidy` produces no changes
- [ ] No stuttering in type names
- [ ] Error messages: lowercase, no punctuation, wrapped with context

## Closeout Augmentations

During task execution Phase 7 (Closeout), add a CHANGELOG entry for the completed work:

- Add a summary line under the `## Current` header in `CHANGELOG.md`
- Create the `## Current` header if it does not exist (insert it after `# Changelog`)
- Entry format: `- [Description of change] (#[issue-number])`

## Release Augmentations

During a release session, apply these Go-specific conventions:

- **Tag format**: Must use `v` prefix (e.g., `v0.2.0`). Go module tooling requires this.
- **Validation**: Run `go vet ./...` and `go test ./tests/...` before tagging.
- **Post-release**: After pushing the tag, the Go module proxy will index the new version automatically. Optionally warm the proxy cache:

```bash
GOPROXY=proxy.golang.org go list -m github.com/owner/project@<version>
```

## Relevant Skills

Load these skills for Go library development:

- **tau:go-patterns** - interface design, error handling, package structure, configuration lifecycle
- **tau:tau-core-admin** - if contributing to the tau-core library specifically (provider/protocol patterns, test organization)
