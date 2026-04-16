# Concurrency

`sync.WaitGroup.Go` (Go 1.25), context propagation rules, the lifecycle
Coordinator pattern, graceful shutdown with timeout, streaming response
channels, and retry with context cancellation.

## Contents

1. [`sync.WaitGroup.Go`](#syncwaitgroupgo)
2. [Context propagation rules](#context-propagation-rules)
3. [Lifecycle Coordinator pattern](#lifecycle-coordinator-pattern)
4. [Graceful shutdown with timeout](#graceful-shutdown-with-timeout)
5. [Streaming response channels](#streaming-response-channels)
6. [Retry and backoff with context cancellation](#retry-and-backoff-with-context-cancellation)

## `sync.WaitGroup.Go`

Go 1.25 added `wg.Go(func())` — a single call that increments the wait
group, spawns a goroutine, and decrements on return.

Old pattern (pre-1.25):

```go
wg.Add(1)
go func() {
    defer wg.Done()
    work()
}()
```

New pattern:

```go
wg.Go(work)
```

Source: `/home/jaime/code/agent-lab/pkg/lifecycle/lifecycle.go:43-51`

Use `wg.Go` in all new code. Keep the old three-line pattern only when
maintaining pre-1.25 codebases that can't move forward.

## Context propagation rules

Rules:

- Every function that performs I/O or blocks accepts `ctx context.Context`
  as the first parameter.
- Background goroutines receive a context that is cancelled on shutdown
  (typically derived from a lifecycle Coordinator's context).
- Never wrap a context with an unrelated parent (except in tests where you
  want to control timeouts independently).
- Never store context in a struct field — **except** for lifecycle
  coordinators where holding a context is the type's entire purpose.

The Coordinator exception:

```go
type Coordinator struct {
    ctx    context.Context
    cancel context.CancelFunc
    // ...
}
```

Source: `/home/jaime/code/agent-lab/pkg/lifecycle/lifecycle.go:18-25`

This is justified because `Coordinator`'s purpose is to hold and distribute
a cancellable context. All other functions thread `ctx` as a parameter:

```go
func (a *agent) Chat(ctx context.Context, messages []protocol.Message, opts ...map[string]any) (*response.Response, error)
```

Source: `/home/jaime/tau/agent/agent.go:117`

## Lifecycle Coordinator pattern

A Coordinator manages startup hooks, shutdown hooks, and readiness state. It
provides a shared cancellable context that subsystems use to detect
shutdown.

Full implementation:

```go
package lifecycle

import (
    "context"
    "fmt"
    "sync"
    "time"
)

type ReadinessChecker interface {
    Ready() bool
}

type Coordinator struct {
    ctx        context.Context
    cancel     context.CancelFunc
    startupWg  sync.WaitGroup
    shutdownWg sync.WaitGroup
    ready      bool
    readyMu    sync.RWMutex
}

func New() *Coordinator {
    ctx, cancel := context.WithCancel(context.Background())
    return &Coordinator{ctx: ctx, cancel: cancel}
}

func (c *Coordinator) Context() context.Context { return c.ctx }

// OnStartup registers a function to run concurrently during startup.
// All registered functions must complete before WaitForStartup returns.
func (c *Coordinator) OnStartup(fn func()) {
    c.startupWg.Go(fn)
}

// OnShutdown registers a function to run concurrently during shutdown.
// Functions should wait for Context().Done() before performing cleanup.
func (c *Coordinator) OnShutdown(fn func()) {
    c.shutdownWg.Go(fn)
}

func (c *Coordinator) Ready() bool {
    c.readyMu.RLock()
    defer c.readyMu.RUnlock()
    return c.ready
}

func (c *Coordinator) WaitForStartup() {
    c.startupWg.Wait()
    c.readyMu.Lock()
    c.ready = true
    c.readyMu.Unlock()
}
```

Source: `/home/jaime/code/agent-lab/pkg/lifecycle/lifecycle.go:1-66`

Usage in a `cmd/server` flow:

```go
lc := lifecycle.New()

// Each subsystem registers its own startup work and shutdown hook
db.Start(lc)
storage.Start(lc)
httpServer.Start(lc)

lc.WaitForStartup()  // blocks until all OnStartup hooks finish

// Wait for signal
<-signalChan

// Cancel context, run all OnShutdown hooks, with timeout
if err := lc.Shutdown(cfg.ShutdownTimeoutDuration()); err != nil {
    log.Fatal("shutdown failed:", err)
}
```

The pattern decouples "what should shut down" from "when to shut down". Any
number of subsystems can register independently, and the Coordinator owns
the orchestration.

## Graceful shutdown with timeout

`Shutdown` cancels the context, waits for all `OnShutdown` hooks to finish,
or returns an error if the timeout elapses first:

```go
func (c *Coordinator) Shutdown(timeout time.Duration) error {
    c.cancel()

    done := make(chan struct{})
    go func() {
        c.shutdownWg.Wait()
        close(done)
    }()

    select {
    case <-done:
        return nil
    case <-time.After(timeout):
        return fmt.Errorf("shutdown timeout after %v", timeout)
    }
}
```

Source: `/home/jaime/code/agent-lab/pkg/lifecycle/lifecycle.go:68-85`

Pattern:

- **Cancel context first.** This signals every blocked operation to return.
- **Wait with a deadline, not forever.** A misbehaving subsystem must not
  hold the process indefinitely.
- **Return an error on timeout.** The caller (typically `cmd/server/main`)
  decides whether to log, force-exit, or wait further.

Signal handling at the top level:

```go
func main() {
    cfg, err := config.Load()
    if err != nil {
        log.Fatal("config load failed:", err)
    }

    srv, err := NewServer(cfg)
    if err != nil {
        log.Fatal("service init failed:", err)
    }

    if err := srv.Start(); err != nil {
        log.Fatal("service start failed:", err)
    }

    sigChan := make(chan os.Signal, 1)
    signal.Notify(sigChan, os.Interrupt, syscall.SIGTERM)
    <-sigChan

    if err := srv.Shutdown(cfg.ShutdownTimeoutDuration()); err != nil {
        log.Fatal("shutdown failed:", err)
    }

    log.Println("service stopped gracefully")
}
```

Source: `/home/jaime/code/herald/cmd/server/main.go`

## Streaming response channels

When returning a stream of values, use a receive-only channel that the
producer closes when finished:

```go
func (a *agent) ChatStream(
    ctx context.Context,
    messages []protocol.Message,
    opts ...map[string]any,
) (<-chan *response.StreamingResponse, error) {
    options := a.mergeOptions(protocol.Chat, opts...)
    options["stream"] = true

    req := request.NewChat(a.provider, a.fmt, a.model, messages, options)

    return a.client.ExecuteStream(ctx, req)
}
```

Source: `/home/jaime/tau/agent/agent.go:139-146`

Rules:

- **Return `<-chan T`** (receive-only). The caller cannot send.
- **Producer closes the channel** when the stream ends. `defer
  close(channel)` in the producing goroutine guarantees closure on every
  return path.
- **Producer respects context cancellation.** Inside the send loop, select
  on `ctx.Done()` so cancelling the context unblocks any pending send.
- **Setup errors return immediately.** A failed setup is `(nil, err)`; a
  stream error is delivered on the channel as part of the element type.

Consumer pattern:

```go
ch, err := agent.ChatStream(ctx, msgs)
if err != nil {
    return err
}
for chunk := range ch {
    // process chunk; loop ends when producer closes the channel
}
```

## Retry and backoff with context cancellation

Retry logic must check context cancellation at every wait point:

```go
func doWithRetry[T any](
    ctx context.Context,
    cfg config.RetryConfig,
    operation func(context.Context) (T, error),
) (T, error) {
    var result T
    var lastErr error

    for attempt := 0; attempt <= cfg.MaxRetries; attempt++ {
        if err := ctx.Err(); err != nil {
            return result, fmt.Errorf("operation cancelled: %w", err)
        }

        result, lastErr = operation(ctx)
        if lastErr == nil {
            return result, nil
        }

        if !isRetryableError(lastErr) {
            return result, lastErr
        }

        if attempt < cfg.MaxRetries {
            delay := calculateBackoff(attempt, cfg)

            select {
            case <-time.After(delay):
            case <-ctx.Done():
                return result, fmt.Errorf("operation cancelled during backoff: %w", ctx.Err())
            }
        }
    }

    return result, fmt.Errorf("max retries (%d) exceeded: %w", cfg.MaxRetries, lastErr)
}
```

Source: `/home/jaime/tau/agent/client/retry.go:104-138`

Key points:

- **Check `ctx.Err()` at the top of each iteration.** Fail fast if the
  context is already cancelled.
- **`select` over `time.After` and `ctx.Done()`** in the backoff wait —
  cancellation must win over the timer.
- **Wrap the final error with retry context.** "max retries (N) exceeded:
  %w" tells the caller both that retries happened and what the underlying
  error was.
- **Generic `[T any]`** lets one retry function serve every operation
  shape.

The backoff calculator uses Go 1.21 builtins:

```go
func calculateBackoff(attempt int, cfg config.RetryConfig) time.Duration {
    maxAttempt := min(attempt, 10)
    delay := cfg.InitialBackoffDuration() * time.Duration(1<<uint(maxAttempt))
    if cfg.Jitter {
        jitterRange := delay / 4
        jitter := time.Duration(rand.Int63n(int64(jitterRange)*2)) - jitterRange
        delay += jitter
    }
    return min(delay, cfg.MaxBackoffDuration())
}
```

Source: `/home/jaime/tau/agent/client/retry.go:84-96`

See [error-handling.md#retry-classification](error-handling.md#retry-classification)
for `isRetryableError`. See [modern-idioms.md#min--max-builtins](modern-idioms.md#min--max-builtins)
for the `min` calls.
