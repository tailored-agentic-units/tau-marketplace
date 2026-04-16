# Registry / Factory Pattern

A pluggable extension pattern: a core package defines an interface, and
implementations register themselves under string names so consumers can
select implementations from configuration without static imports.

## When to use

This pattern fits when:

- The core package defines an interface but **cannot know all
  implementations** at compile time.
- Implementations live in **separate sub-modules** so consumers can opt into
  exactly the implementations they need.
- **Configuration selects implementation by string name** (e.g.,
  `provider.name = "azure"` in JSON).

In the tau ecosystem, two contexts use this pattern:

- `format` core + `format/openai`, `format/converse` implementations.
- `provider` core + `provider/azure`, `provider/bedrock`, `provider/ollama`
  implementations.

Skip the pattern when:

- All implementations live in the same package — just call the constructor.
- The set of implementations is fixed and small — a `switch` in the
  constructor is clearer.

## Registry structure

Full implementation of the provider registry:

```go
package provider

import (
    "fmt"
    "sync"

    "github.com/tailored-agentic-units/protocol/config"
)

// Factory is a function that creates a Provider from configuration.
// Provider implementations register their factory function to enable dynamic provider creation.
type Factory func(c *config.ProviderConfig) (Provider, error)

// registry maintains the global provider factory registry.
// It is thread-safe for concurrent registration and provider creation.
type registry struct {
    factories map[string]Factory
    mu        sync.RWMutex
}

// register is the global provider factory registry.
var register = &registry{
    factories: make(map[string]Factory),
}

// Register registers a provider factory with the given name.
// Thread-safe for concurrent registration.
func Register(name string, factory Factory) {
    register.mu.Lock()
    defer register.mu.Unlock()
    register.factories[name] = factory
}

// Create creates a Provider instance from configuration.
// Returns an error if the provider name is not registered.
// Thread-safe for concurrent provider creation.
func Create(c *config.ProviderConfig) (Provider, error) {
    register.mu.RLock()
    factory, exists := register.factories[c.Name]
    register.mu.RUnlock()

    if !exists {
        return nil, fmt.Errorf("unknown provider: %s", c.Name)
    }

    return factory(c)
}

// ListProviders returns a list of all registered provider names.
// Thread-safe for concurrent access.
func ListProviders() []string {
    register.mu.RLock()
    defer register.mu.RUnlock()

    names := make([]string, 0, len(register.factories))
    for name := range register.factories {
        names = append(names, name)
    }
    return names
}
```

Source: `/home/jaime/tau/provider/registry.go`

Anatomy:

- **`Factory` typedef** — the constructor signature implementations must
  match.
- **`registry` struct** — a `map[string]Factory` plus `sync.RWMutex`. The
  type is unexported.
- **`register` package-scope variable** — the global registry instance,
  initialized inline.
- **`Register(name, factory)`** — write lock on the mutex.
- **`Create(c)`** — read lock to look up the factory, then call it
  outside the lock so factories can do their own locking without deadlock
  risk.
- **`List*()`** — read lock and copy the names into a fresh slice (never
  return the internal map).

## Factory function signatures

Two styles are in use across the tau ecosystem.

### Style A — config-parameterized (provider)

```go
type Factory func(c *config.ProviderConfig) (Provider, error)
```

Source: `/home/jaime/tau/provider/registry.go:11-12`

Use when different implementations need different runtime configuration.
The factory inspects the config (`c.Options["deployment"]`,
`c.Options["api_version"]`, etc.) and returns a configured provider.

### Style B — no-arg (format)

```go
type Factory func() (Format, error)
```

Source: `/home/jaime/tau/format/registry.go:9`

Use when implementations are stateless and identical from one instance to
the next. The factory is a constructor for a singleton-ish value.

Choose A when implementation behavior depends on per-instance configuration;
choose B when one instance is interchangeable with any other.

## Registration patterns

### Explicit registration (tau convention)

Implementations expose a `Register()` function that the application calls:

```go
package openai

import (
    "github.com/tailored-agentic-units/format"
    // ...
)

// Register registers the OpenAI format with the global format registry.
func Register() {
    format.Register("openai", Factory)
}

// Factory creates a new Format instance for use with the format registry.
func Factory() (format.Format, error) {
    return &Format{}, nil
}
```

Source: `/home/jaime/tau/format/openai/openai.go:12-20`

Caller:

```go
import (
    "github.com/tailored-agentic-units/format/openai"
    "github.com/tailored-agentic-units/format/converse"
    // ...
)

func main() {
    openai.Register()
    converse.Register()
    // ...
}
```

Trade-off: explicit. The caller knows exactly which implementations are
active. Slightly more boilerplate.

### `init()`-time registration (classic Go plugin pattern)

```go
// In format/openai/openai.go
func init() {
    format.Register("openai", Factory)
}

// Caller:
import _ "github.com/tailored-agentic-units/format/openai"  // blank import
```

Trade-off: automatic. The blank import signals "activate this
implementation". Hidden side effects can surprise readers and complicate
debugging.

**Recommendation**: tau prefers explicit registration. Follow that
convention unless the package is intended for drop-in activation patterns
where the import is the registration.

## Thread safety

The registry uses `sync.RWMutex`:

- **`RLock`** for reads (`Create`, `List`) — many concurrent consumers can
  proceed simultaneously.
- **`Lock`** for writes (`Register`) — only one registration at a time.

Always lock, even if you "know" registration happens at `init()` time.
Tests, alternate compositions, and dynamic plugin loaders break that
assumption. The cost of the mutex is negligible.

The factory function itself runs **outside** the registry lock. This is
intentional: factories may take time, do I/O, or acquire other locks. Holding
the registry's read lock during factory execution would serialize all
constructions on the whole registry.

## Usage from application

End-to-end flow:

```go
// main.go
openai.Register()
converse.Register()
azure.Register()
bedrock.Register()
ollama.Register()

// Later, from config:
cfg, _ := config.LoadAgentConfig("config.json")
p, err := provider.Create(cfg.Provider)  // resolves cfg.Provider.Name → factory
f, err := format.Create(cfg.Format)      // resolves cfg.Format → factory
a := agent.New(cfg, p, f)
```

The registry is what enables configuration-driven composition: the core
`provider` package never imports `provider/azure`, but at runtime
`provider.Create(cfg.Provider)` finds the Azure factory and calls it.

## Anti-patterns

- **Registry without a mutex.** Concurrent test startup will produce
  intermittent failures that are hard to reproduce.
- **Returning the internal map from `List*()`.** Callers could mutate it.
  Always copy the keys into a fresh slice.
- **Factories that take non-config dependencies.** Threading
  `(cfg, logger, db)` through factory signatures couples the registration
  side to the application's wiring order. Factories should be pure given
  configuration; logging and other cross-cutting concerns belong inside the
  factory or in a wrapper around it.
- **Test isolation problems with the global registry.** If tests need
  isolation, expose a `newRegistry()` helper that returns a fresh registry
  for the test to use. The global registry stays as the default for normal
  callers.
