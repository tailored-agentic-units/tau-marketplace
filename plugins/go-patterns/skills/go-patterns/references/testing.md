# Testing

Black-box testing layout, table-driven test shape, when to stand up a real
HTTP server vs mock an interface, where mock packages live, how to assert on
concurrent behavior without `time.Sleep`, and helper conventions.

## Contents

1. [Black-box vs white-box](#black-box-vs-white-box)
2. [`tests/` directory mirroring](#tests-directory-mirroring)
3. [Table-driven test shape](#table-driven-test-shape)
4. [`httptest.NewServer` for real HTTP](#httptestnewserver-for-real-http)
5. [Mock packages](#mock-packages)
6. [Concurrency assertions with `atomic.Int32`](#concurrency-assertions-with-atomicint32)
7. [Test helpers and fixtures](#test-helpers-and-fixtures)

## Black-box vs white-box

**Default**: black-box. Test files declare `package <pkg>_test` and import
the package they test through its public API.

```go
package agent_test

import (
    "context"
    "errors"
    "strings"
    "testing"

    "github.com/tailored-agentic-units/agent"
    "github.com/tailored-agentic-units/agent/mock"
    "github.com/tailored-agentic-units/format"
    "github.com/tailored-agentic-units/protocol"
    "github.com/tailored-agentic-units/protocol/config"
)
```

Source: `/home/jaime/tau/agent/tests/agent_test.go:1-15`

**Exception**: white-box (`package <pkg>`, no `_test` suffix) is reserved for
unexported helpers that cannot be exercised through the public API. Place
these in the same directory as source.

Why black-box by default:

- Tests double as a contract consumer. They break when the public API
  breaks, not when internals change.
- Refactoring is safer — you know existing callers will still compile.
- The package's testable surface equals its public surface.

## `tests/` directory mirroring

Tests live under `tests/`, not alongside source. The tree under `tests/`
matches the source tree 1:1:

```
agent/
├── agent.go
├── client/
│   ├── client.go
│   └── retry.go
├── mock/
│   ├── agent.go
│   └── helpers.go
└── tests/
    ├── agent_test.go         # for agent.go
    ├── client/
    │   ├── client_test.go
    │   └── retry_test.go
    └── mock/
        └── agent_test.go
```

Conventions:

- One source file → one test file with the same base name.
- Shared helpers live in `tests/testutil.go` at the top level — flat, never
  a nested `tests/testutil/` package.
- The test directory is excluded from production builds because `_test.go`
  suffix triggers Go's test-only compilation.

## Table-driven test shape

Use anonymous struct slices for parameterized cases:

```go
func TestRetryClassification(t *testing.T) {
    tests := []struct {
        name      string
        err       error
        wantRetry bool
    }{
        {"nil error", nil, false},
        {"context canceled", context.Canceled, false},
        {"context deadline exceeded", context.DeadlineExceeded, false},
        {"HTTP 429", &HTTPStatusError{StatusCode: 429}, true},
        {"HTTP 502", &HTTPStatusError{StatusCode: 502}, true},
        {"HTTP 400", &HTTPStatusError{StatusCode: 400}, false},
        {"HTTP 500", &HTTPStatusError{StatusCode: 500}, false},
    }
    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            if got := isRetryableError(tt.err); got != tt.wantRetry {
                t.Errorf("got %v, want %v", got, tt.wantRetry)
            }
        })
    }
}
```

Conventions:

- **Anonymous struct defined inline.** Lifting it out is rarely worth the
  ceremony.
- **`name` field first** for readable case-by-case test output.
- **`t.Run(tt.name, ...)`** so individual subtests report independently and
  can be filtered with `-run`.
- **`want*` prefix for expected values** (`wantRetry`, `wantErr`,
  `wantContent`).
- **No shared mutable state** across iterations. Each row stands on its own.

For test cases that need setup beyond the inline struct, lift the setup into
a per-row helper rather than threading shared state.

## `httptest.NewServer` for real HTTP

When the code under test makes HTTP calls, stand up a real
`httptest.NewServer` instead of mocking `http.Client`:

```go
func TestClient_Execute(t *testing.T) {
    server := httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        w.Header().Set("Content-Type", "application/json")
        w.WriteHeader(http.StatusOK)
        w.Write([]byte(`{"choices":[{"message":{"role":"assistant","content":"hi"},"finish_reason":"stop"}]}`))
    }))
    defer server.Close()

    cfg := config.DefaultClientConfig()
    cfg.BaseURL = server.URL

    // construct client, exercise, assert
}
```

Why this beats mocking `http.Client`:

- Exercises the **full stack**: marshaling, transport, connection pooling,
  retry logic, response parsing.
- The fake server is a real `*http.Server`; it behaves like production
  infrastructure with respect to timeouts, cancellation, and concurrent
  requests.
- The test asserts on **observable behavior**, not on which methods got
  called on a mock.

Reserve interface mocking for **logical** dependencies (Provider, Format,
Agent), not transport-layer concerns (HTTP, database connections).

## Mock packages

For interfaces that represent injected dependencies, provide a `mock/`
sub-package co-located with the interface owner:

```
agent/
├── agent.go
├── mock/
│   ├── agent.go      # MockAgent struct + WithX option functions
│   ├── client.go     # MockClient
│   ├── format.go     # MockFormat
│   ├── provider.go   # MockProvider
│   └── helpers.go    # NewSimpleChatAgent, NewStreamingChatAgent, ...
```

Source: `/home/jaime/tau/agent/mock/`

Mock construction uses functional options for the same reason error
construction does — many call sites need different combinations of stubbed
behavior:

```go
func NewSimpleChatAgent(id string, content string) *MockAgent {
    chatResponse := &response.Response{
        Role: "assistant",
        Content: []response.ContentBlock{
            response.TextBlock{Text: content},
        },
    }

    return NewMockAgent(
        WithAgentID(id),
        WithChatResponse(chatResponse, nil),
    )
}

func NewFailingAgent(id string, err error) *MockAgent {
    return NewMockAgent(
        WithAgentID(id),
        WithChatResponse(nil, err),
        WithVisionResponse(nil, err),
        WithToolsResponse(nil, err),
        WithEmbeddingsResponse(nil, err),
        WithStreamResponses(nil, err),
    )
}
```

Source: `/home/jaime/tau/agent/mock/helpers.go:9-21, 105-114`

Why co-located:

- The mock implements the interface from the same package. When the interface
  changes, the mock changes in the same commit.
- Tests in any other package can `import "github.com/.../agent/mock"` and
  build realistic stubs.

## Concurrency assertions with `atomic.Int32`

When tests verify goroutine ordering or invocation counts, use
`sync/atomic` counters rather than `time.Sleep` polling or channel
gymnastics:

```go
func TestCoordinator_OnShutdown_Multiple(t *testing.T) {
    c := lifecycle.New()

    var calls atomic.Int32
    for range 3 {
        c.OnShutdown(func() {
            <-c.Context().Done()
            calls.Add(1)
        })
    }

    if err := c.Shutdown(time.Second); err != nil {
        t.Fatalf("shutdown: %v", err)
    }

    if got := calls.Load(); got != 3 {
        t.Errorf("got %d shutdown calls, want 3", got)
    }
}
```

Conventions:

- **`atomic.Int32` or `atomic.Int64`** — never `int` with manual locking.
- **`for range n`** to register multiple hooks (Go 1.22 idiom; see
  [modern-idioms.md](modern-idioms.md)).
- **`Load()`** for the assertion read; **`Add(1)`** for the increment.
- **No `time.Sleep`.** Synchronize through channels, `WaitGroup`, or the
  primitive being tested (here, `c.Shutdown()` blocks until hooks return).

If an assertion really needs a "wait until at least once", use `Load() >= 1`
rather than sleeping.

## Test helpers and fixtures

Fixture placement:

- **Inline** small fixtures (a few lines) inside the test function.
- **File-scope helper** for shared fixtures that span multiple tests in one
  file:

```go
func newTestConfig() *config.AgentConfig {
    return &config.AgentConfig{
        Name: "test-agent",
        Client: &config.ClientConfig{
            Timeout:            "30s",
            ConnectionTimeout:  "10s",
            ConnectionPoolSize: 10,
            Retry: config.RetryConfig{MaxRetries: 0},
        },
        Provider: &config.ProviderConfig{
            Name:    "mock-provider",
            BaseURL: "http://localhost:11434",
        },
        Model: &config.ModelConfig{
            Name: "test-model",
            Capabilities: map[string]map[string]any{
                "chat": {"temperature": 0.7},
            },
        },
    }
}
```

Source: `/home/jaime/tau/agent/tests/agent_test.go:17-39`

- **`tests/testdata/`** for large binary fixtures (images, golden files). Go
  excludes this directory from builds automatically.

Helper naming:

- `newXxx` for constructors returning fresh state per test (no shared
  references between tests).
- `assertXxx` for assertion helpers — call `t.Helper()` inside so failure
  reports point at the test, not the helper.
- `t.Cleanup(func() { ... })` instead of `setup`/`teardown` ceremonies.
  Cleanups run in reverse registration order, just before the test
  function returns.
