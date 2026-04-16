# Constructors

`New()` shapes, when to return interfaces vs concretes, why struct configs
beat functional options for production constructors, the narrow place where
variadic options are appropriate, and where the composition root sits in each
project type.

## Contents

1. [`New()` returns interface or concrete](#new-returns-interface-or-concrete)
2. [Struct configs over functional options](#struct-configs-over-functional-options)
3. [Variadic options: when appropriate](#variadic-options-when-appropriate)
4. [Runtime options as `map[string]any`](#runtime-options-as-mapstringany)
5. [Composition root](#composition-root)
6. [Anti-patterns](#anti-patterns)

## `New()` returns interface or concrete

Decision rule:

- If the package exports an interface for the type → `New` returns the
  interface.
- If the type is pure data, an embedding base, or otherwise has no
  abstraction need → `New` returns a `*Concrete` (or value).

### Interface return

```go
func New(cfg *config.AgentConfig, p provider.Provider, f format.Format) Agent {
    m := model.New(cfg.Model)
    c := client.New(cfg.Client)
    return &agent{
        id:       uuid.Must(uuid.NewV7()).String(),
        client:   c,
        provider: p,
        fmt:      f,
        model:    m,
    }
}
```

Source: `/home/jaime/tau/agent/agent.go:81-92`

Returning the `Agent` interface means callers can't reach into private
fields, which makes future internal changes safe. See
[package-design.md#unexported-structs-returned-as-interfaces](package-design.md#unexported-structs-returned-as-interfaces).

### Concrete return

```go
func NewBaseProvider(name, baseURL string) *BaseProvider {
    return &BaseProvider{
        name:    name,
        baseURL: baseURL,
    }
}
```

Source: `/home/jaime/tau/provider/base.go:13-18`

`BaseProvider` is meant to be **embedded** by concrete providers, not
substituted by them. There is no Provider-shaped interface to return; the
package exports the struct directly because that's its purpose.

## Struct configs over functional options

For production constructors, take a struct config parameter:

```go
func New(cfg *config.AgentConfig, p provider.Provider, f format.Format) Agent
```

Why struct configs win:

- **Self-documenting.** The struct definition lists every input. Reading
  `AgentConfig` tells you exactly what the agent needs.
- **Defaults in one place.** `DefaultAgentConfig()` provides sensible
  starting values; callers override only what they care about.
- **Serializable.** The same struct loads from JSON, yaml, env vars, etc. A
  functional-options API has no equivalent.
- **Required vs optional is visible.** Pointer fields with `omitempty` are
  optional; required fields have no `omitempty` and zero values fail
  validation.

The functional-options anti-pattern for constructors:

```go
// BAD
func NewAgent(opts ...AgentOption) Agent {
    a := &agent{}
    for _, opt := range opts { opt(a) }
    // ... now what's required? what's optional? what's the default?
    return a
}
```

This pattern hides the required surface, scatters defaults across many
`WithX` functions, and produces no serializable representation. Use struct
configs in production code.

The one place functional options belong: error enrichment. See
[error-handling.md#functional-options-for-enrichment](error-handling.md#functional-options-for-enrichment).

## Variadic options: when appropriate

Variadic option functions are right when:

- There are 0–N genuinely optional parameters with no meaningful defaults.
- Call sites benefit from named-parameter clarity.
- A struct config would be mostly empty fields.

The clearest example is mock construction:

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
```

Source: `/home/jaime/tau/agent/mock/helpers.go:9-21`

A `MockAgent` may stub out chat, vision, tools, embeddings, or streaming —
but typically only one or two at a time. A `MockAgentConfig` struct with all
five fields would mostly be empty. Variadic options express "configure these
specific behaviors" naturally.

Builder-style fluent APIs (`builder.WithProjection(p).Where(c).OrderBy(s)`)
similarly justify variadic-style configuration. Production constructors do
not.

## Runtime options as `map[string]any`

For protocol-level runtime parameters that vary per call (temperature, max
tokens, tool choice), use `map[string]any` rather than a typed struct.

```go
func (a *agent) mergeOptions(proto protocol.Protocol, opts ...map[string]any) map[string]any {
    options := make(map[string]any)
    if modelOpts := a.model.Options[proto]; modelOpts != nil {
        maps.Copy(options, modelOpts)
    }
    if len(opts) > 0 && opts[0] != nil {
        maps.Copy(options, opts[0])
    }
    return options
}
```

Source: `/home/jaime/tau/agent/agent.go:241-251`

Why `map[string]any` and not a typed struct:

- These options pass through to provider APIs that each accept their own
  arbitrary key-value set. Trying to type-check them at the agent layer is
  false economy — every provider has a different valid set.
- Model defaults and per-call overrides merge naturally with `maps.Copy`.
  See [modern-idioms.md#mapscopy](modern-idioms.md#mapscopy).
- Callers who do want type safety can build their own typed wrappers and
  flatten to a map at the call site.

Use this pattern only for genuinely heterogeneous, downstream-defined
options. For inputs the package itself processes, use a typed struct.

## Composition root

The composition root is the single place where concrete dependencies are
wired together. Its location depends on the project type:

- **Library**: there is no composition root. The library accepts interfaces
  and returns interfaces; the consumer composes.
- **Application** (CLI/service): the composition root lives in `cmd/<name>/`.
  Specifically, the `NewServer(cfg)` function or equivalent.

### `Infrastructure` example

The canonical app-level composition pattern is the `Infrastructure` type:

```go
type Infrastructure struct {
    Lifecycle *lifecycle.Coordinator
    Logger    *slog.Logger
    Database  database.System
    Storage   storage.System
}

func New(cfg *config.Config) (*Infrastructure, error) {
    lc := lifecycle.New()
    logger := logging.New(&cfg.Logging)

    db, err := database.New(&cfg.Database, logger)
    if err != nil {
        return nil, fmt.Errorf("database init failed: %w", err)
    }

    store, err := storage.New(&cfg.Storage, logger)
    if err != nil {
        return nil, fmt.Errorf("storage init failed: %w", err)
    }

    return &Infrastructure{
        Lifecycle: lc,
        Logger:    logger,
        Database:  db,
        Storage:   store,
    }, nil
}

func (i *Infrastructure) Start() error {
    if err := i.Database.Start(i.Lifecycle); err != nil {
        return fmt.Errorf("database start failed: %w", err)
    }
    if err := i.Storage.Start(i.Lifecycle); err != nil {
        return fmt.Errorf("storage start failed: %w", err)
    }
    return nil
}
```

Source: `/home/jaime/code/agent-lab/internal/infrastructure/infrastructure.go`

Key decisions:

- **`New(cfg)` constructs but does not start.** Allows callers to inspect or
  swap parts before activating any background work.
- **`Start()` activates each subsystem in order.** Subsystems register their
  shutdown hooks with `Lifecycle` here.
- **Wrapped error messages include subsystem name.** "database init failed"
  is more debuggable than just "init failed".

Higher-level wiring (HTTP routes, domain modules) happens above
`Infrastructure` in `cmd/server/server.go`. The signal-handling main loop is
even higher in `cmd/server/main.go`. See
[concurrency.md#lifecycle-coordinator](concurrency.md#lifecycle-coordinator)
for how `Lifecycle` orchestrates startup and shutdown.

## Anti-patterns

- **Constructors that panic on invalid config.** Return an error so the
  caller can decide; a library that panics is unembeddable.
- **I/O at package init time.** Side effects in `init()` make tests harder
  to control and turn package import order into a runtime dependency.
- **Six-parameter `New(ctx, cfg, db, storage, logger, metrics)`.** Pack into
  a struct or thread the dependencies through an `Infrastructure`-style
  holder.
- **`NewXxx(*Xxx)` that mutates its argument.** If you really need this
  shape (rare), name it `(*Xxx).Init()` so the mutation is visible.
- **Returning a struct from a package that should present an interface.**
  Once exported, callers couple to the struct shape; future changes break
  them.
