# Error Handling

A complete error taxonomy: where errors live, when to use sentinels vs typed
errors, how to enrich errors with functional options, the conventions for
wrapping and classification, the `HTTPStatusError` pattern for retry logic,
and the narrow scope where `errors.Join` is appropriate.

## Contents

1. [`errors.go` file convention](#errorsgo-file-convention)
2. [Sentinel errors](#sentinel-errors)
3. [Typed errors with metadata](#typed-errors-with-metadata)
4. [Functional options for enrichment](#functional-options-for-enrichment)
5. [Wrapping with `%w`](#wrapping-with-w)
6. [Retry classification](#retry-classification)
7. [`HTTPStatusError` pattern](#httpstatuserror-pattern)
8. [`errors.Join` at aggregation boundaries](#errorsjoin-at-aggregation-boundaries)
9. [Error message style](#error-message-style)

## `errors.go` file convention

Any package that defines more than one error type uses an `errors.go` file
holding:

- Sentinel error variables.
- Typed error struct(s) with their methods.
- Functional option types and helpers for enriching typed errors.
- Constructor functions for the typed errors.

Source: `/home/jaime/tau/agent/errors.go` (full file, 162 lines)

The file is the package's complete error vocabulary, in one place. Callers
only need to look here to understand what failures the package signals.

## Sentinel errors

Use `errors.New` for stable identity values that callers compare with
`errors.Is`:

```go
var (
    ErrAgentNotFound  = errors.New("agent not found")
    ErrAgentExists    = errors.New("agent already registered")
    ErrEmptyAgentName = errors.New("agent name is empty")
)
```

Source: `/home/jaime/tau/agent/errors.go:23-28`

Conventions:

- **`Err` prefix** for exported names.
- **Lowercase message**, no period, no trailing context.
- Callers wrap with their own context when they return the error onward.

Use sentinels only when the caller will **branch on identity**. For
one-off errors that no one will catch, just `fmt.Errorf(...)`.

## Typed errors with metadata

When errors carry structured fields (category, ID, timestamp, related entity),
define a struct that implements `error` and `Unwrap`:

```go
type ErrorType string

const (
    ErrorTypeInit ErrorType = "init"
    ErrorTypeLLM  ErrorType = "llm"
)

type AgentError struct {
    Type      ErrorType `json:"type"`
    ID        uuid.UUID `json:"uuid,omitempty"`
    Name      string    `json:"name,omitempty"`
    Code      string    `json:"code,omitempty"`
    Message   string    `json:"message"`
    Cause     error     `json:"-"`
    Client    string    `json:"client,omitempty"`
    Timestamp time.Time `json:"timestamp"`
}

func (e *AgentError) Error() string {
    if e.Client != "" && e.Name != "" {
        return fmt.Sprintf("Agent error [%s/%s]: %s", e.Client, e.Name, e.Message)
    }
    if e.Name != "" {
        return fmt.Sprintf("Agent error [%s]: %s", e.Name, e.Message)
    }
    return fmt.Sprintf("Agent error: %s", e.Message)
}

func (e *AgentError) Unwrap() error {
    return e.Cause
}
```

Source: `/home/jaime/tau/agent/errors.go:30-91`

Structural details to copy:

- **Category enum** distinguishes logical origin (`init` vs `llm`).
- **`Cause` field tagged `json:"-"`** — unwrappable but not serialized
  (cause chain is for runtime introspection, not transport).
- **`Error()` branches on populated context** — show the most specific
  message available rather than padding empty fields.
- **`Unwrap()`** enables `errors.Is(myErr, originalCause)` to traverse the
  chain.

Use a typed error when more than one call site benefits from inspecting the
same structured fields (e.g., metrics that bucket by category).

## Functional options for enrichment

Typed errors with many optional fields use functional options on
construction. This is the **only** place functional options are preferred to
struct configs in this ecosystem (see [constructors.md](constructors.md) for
why constructors prefer struct configs).

```go
type ErrorOption func(*AgentError)

func WithCode(code string) ErrorOption {
    return func(e *AgentError) { e.Code = code }
}

func WithCause(cause error) ErrorOption {
    return func(e *AgentError) { e.Cause = cause }
}

func WithName(name string) ErrorOption {
    return func(e *AgentError) { e.Name = name }
}

func WithAgent(cfg *config.AgentConfig) ErrorOption {
    return func(e *AgentError) {
        // populate e.Client from cfg.Provider.Name + cfg.Model.Name
    }
}

func WithID(id uuid.UUID) ErrorOption {
    return func(e *AgentError) { e.ID = id }
}
```

Source: `/home/jaime/tau/agent/errors.go:93-150`

Construction site:

```go
return nil, NewAgentLLMError("request failed",
    WithCause(err),
    WithAgent(cfg),
    WithID(uuid.Must(uuid.NewV7())),
)
```

Convenience constructors hard-code the category:

```go
func NewAgentInitError(message string, options ...ErrorOption) *AgentError {
    return NewAgentError(ErrorTypeInit, message, options...)
}

func NewAgentLLMError(message string, options ...ErrorOption) *AgentError {
    return NewAgentError(ErrorTypeLLM, message, options...)
}
```

Source: `/home/jaime/tau/agent/errors.go:152-162`

Why functional options work for errors but not for general constructors:
errors are constructed at many sites with different combinations of
available context (sometimes you have a cause, sometimes you have an
agent name, sometimes both). A struct literal with ten fields, most
left empty, reads worse than three named option calls.

## Wrapping with `%w`

Wrap when returning an error from a lower layer. The wrap message describes
**what this function was doing** when it failed:

```go
if err := cfg.validate(); err != nil {
    return fmt.Errorf("finalize config: %w", err)
}
```

Source: `/home/jaime/code/herald/internal/config/config.go:117`

Conventions:

- **Prefix is what the function was doing** ("finalize config", not "config
  finalize failed").
- **No colon duplication** — `%w` adds its own; you write `"prefix: %w"` not
  `"prefix:: %w"`.
- **No trailing punctuation.**
- **Wrap at every layer transition** unless the caller's stack frame already
  identifies the context (e.g., a deferred handler that knows which subsystem
  it represents).

## Retry classification

Use `errors.Is` for sentinel/context comparisons; use `errors.As` for typed
error inspection. Both traverse the wrap chain.

```go
func isRetryableError(err error) bool {
    if err == nil {
        return false
    }

    // Never retry context errors
    if errors.Is(err, context.Canceled) || errors.Is(err, context.DeadlineExceeded) {
        return false
    }

    // Check for HTTP status errors
    var httpErr *HTTPStatusError
    if errors.As(err, &httpErr) {
        return httpErr.StatusCode == 429 ||
            httpErr.StatusCode == 502 ||
            httpErr.StatusCode == 503 ||
            httpErr.StatusCode == 504
    }

    // Check for network operation errors
    var netOpErr *net.OpError
    if errors.As(err, &netOpErr) {
        return true
    }

    // Check for DNS errors
    var dnsErr *net.DNSError
    if errors.As(err, &dnsErr) {
        return dnsErr.Temporary() || dnsErr.Timeout()
    }

    // URL errors — unwrap and recursively check
    var urlErr *url.Error
    if errors.As(err, &urlErr) {
        return isRetryableError(urlErr.Err)
    }

    return false
}
```

Source: `/home/jaime/tau/agent/client/retry.go:40-78`

Teaching points:

- `errors.Is` checks identity (same sentinel value somewhere in the chain).
- `errors.As` walks the chain looking for an error of the requested type and
  populates the target if found.
- When the standard `Unwrap()` chain isn't enough — e.g., `url.Error` wraps
  the underlying error in `.Err` rather than returning it from `Unwrap()` —
  recurse explicitly.

## `HTTPStatusError` pattern

When HTTP status matters to callers (for retry, for surfacing user-visible
errors, for metrics), wrap status codes in a typed error:

```go
type HTTPStatusError struct {
    StatusCode int
    Status     string
    Body       []byte
}

func (e *HTTPStatusError) Error() string {
    if len(e.Body) > 0 {
        return fmt.Sprintf("HTTP %d: %s - %s", e.StatusCode, e.Status, string(e.Body))
    }
    return fmt.Sprintf("HTTP %d: %s", e.StatusCode, e.Status)
}
```

Source: `/home/jaime/tau/agent/client/retry.go:15-28`

Why this beats `fmt.Errorf("HTTP %d", code)`:

- Callers branch on the integer status via `errors.As` rather than parsing
  strings.
- The body is preserved structurally, not concatenated into a message.
- The retry classifier above shows the payoff: one `errors.As` extracts the
  whole struct.

## `errors.Join` at aggregation boundaries

`errors.Join` collapses multiple unrelated errors into a single error.
Reserve it for **aggregation boundaries** — places where multiple
independent operations may fail and the caller wants to see all failures, not
just the first.

Good — collecting per-item validation errors:

```go
func validateAll(items []Item) error {
    var errs []error
    for _, item := range items {
        if err := item.Validate(); err != nil {
            errs = append(errs, fmt.Errorf("item %s: %w", item.ID, err))
        }
    }
    return errors.Join(errs...) // returns nil if errs is empty
}
```

Other valid uses:

- Batch processors where each item is independent.
- Multi-resource shutdown where you want to log every cleanup failure.
- Parallel fan-out where independent goroutines may each fail.

Bad — sequential operations with distinct semantics:

```go
err1 := step1()
err2 := step2()
return errors.Join(err1, err2) // NO — return first, or wrap properly
```

`errors.Join` hides causation. Sequential operations should return the first
failure (so the caller knows which step broke) or wrap one with the other
(so the chain is preserved). Reach for `Join` only when the failures are
**genuinely independent**.

## Error message style

- **Lowercase first letter.**
- **No trailing punctuation.**
- **Don't say "error" or "failed"** — those are obvious from context. "config
  file not found" beats "failed to find config file".
- **Describe what happened, not what was attempted.** "connection refused" is
  better than "failed to connect".
- **Keep messages stable.** Callers should compare errors via `errors.Is` /
  `errors.As`, but in practice some will grep logs. Don't churn message text
  across releases unless you have a reason.
