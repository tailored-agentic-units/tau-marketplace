# Package Design

How packages name themselves, depend on each other, expose interfaces, hide
implementations, share behavior through embedding, and present semantic
accessors instead of leaking internal structure.

## Contents

1. [Package naming](#package-naming)
2. [Dependency direction](#dependency-direction)
3. [Interfaces owned by the consumer](#interfaces-owned-by-the-consumer)
4. [Unexported structs returned as interfaces](#unexported-structs-returned-as-interfaces)
5. [Embedding for shared behavior](#embedding-for-shared-behavior)
6. [Semantic getters](#semantic-getters)
7. [Cross-package boundaries](#cross-package-boundaries)

## Package naming

Use singular, lowercase nouns. The name describes what the package
**provides**, not what it **contains**.

Good: `agent`, `client`, `provider`, `format`, `protocol`, `response`,
`registry`, `request`, `streaming`.

Avoid: `types`, `utils`, `common`, `helpers`, `shared`, `agents` (plural),
`agent_pkg` (underscore). These names indicate the package has no clear
purpose — split it or rename it.

Sub-packages within a domain use concern-specific names that compose with the
parent: `agent/client`, `agent/request`, `agent/registry`, `agent/mock`.

## Dependency direction

Dependencies flow in one direction only: from higher (composing) layers to
lower (foundational) layers.

```
protocol  ←  format, provider  ←  agent  ←  orchestrate
```

The tau ecosystem demonstrates this:

- `protocol` defines `Protocol`, `Message`, `ContentBlock`,
  `StreamReader`, and the `config` types.
- `format` and `provider` import `protocol` to satisfy its interfaces.
- `agent` composes a `format.Format`, a `provider.Provider`, and an HTTP
  client to satisfy the `Agent` interface it defines for itself.
- `orchestrate` composes multiple `agent.Agent` instances and adds
  message routing on top.

Rules:

- **Lower layers never import higher layers.** If you find yourself wanting
  the reverse, you have a layering mistake or a missing interface.
- **Shared types live in the lowest layer that needs them.** `Message` lives
  in `protocol` because every layer above uses it.
- **Use dependency inversion when needed.** When a lower layer needs behavior
  from a higher layer, the lower layer defines an interface and the higher
  layer provides the implementation.

See [project-layout.md#depth-nuance-sub-packages-vs-sub-modules](project-layout.md#depth-nuance-sub-packages-vs-sub-modules)
for how to extend a package family across sub-modules without breaking the
single-module depth budget.

## Interfaces owned by the consumer

The package that **uses** the interface is the package that **defines** it.
This is the inverse of how some other ecosystems work, where interfaces are
"part of" the implementation package.

Example: `agent` consumes a provider, so `agent` would own a Provider
interface if the relationship were one-to-one. In practice, since multiple
provider implementations exist as separate sub-modules and registry creates
them dynamically, the interface lives in the foundational `provider` package
that all implementations import. The principle still holds: the place that
needs the contract is the place where the contract is shaped.

For interfaces with a single consumer, define them in the consumer:

```go
// agent/agent.go — agent is the consumer
type Agent interface {
    ID() string
    Client() client.Client
    Provider() provider.Provider
    Format() format.Format
    Model() *model.Model
    Chat(ctx context.Context, messages []protocol.Message, opts ...map[string]any) (*response.Response, error)
    // ...
}
```

Source: `/home/jaime/tau/agent/agent.go:26-67`

The interface lives in `agent/`, not in some `interfaces/` package. Consumers
that need to mock an Agent use `agent/mock/`.

## Unexported structs returned as interfaces

When a type has behavior plus private state, the struct is unexported and the
constructor returns the exported interface:

```go
// Exported interface
type Agent interface { /* ... */ }

// Unexported concrete
type agent struct {
    id       string
    client   client.Client
    provider provider.Provider
    fmt      format.Format
    model    *model.Model
}

// Constructor returns the interface
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

Source: `/home/jaime/tau/agent/agent.go:69-92`

Why:

- Callers cannot reach into struct fields, so internal layout can change
  without breaking consumers.
- Mock substitution works through the interface.
- The package's testable surface is exactly its public API.

When **not** to apply this pattern: pure data types (DTOs, response objects,
config structs) remain exported structs. `response.Response` is an exported
struct because callers legitimately read its fields (often through semantic
getters — see below).

## Embedding for shared behavior

When several sibling implementations share identical fields and methods,
factor them into a base struct and embed it:

```go
package provider

type BaseProvider struct {
    name    string
    baseURL string
}

func NewBaseProvider(name, baseURL string) *BaseProvider {
    return &BaseProvider{name: name, baseURL: baseURL}
}

func (p *BaseProvider) Name() string    { return p.name }
func (p *BaseProvider) BaseURL() string { return p.baseURL }
```

Source: `/home/jaime/tau/provider/base.go`

Each concrete provider embeds it:

```go
type AzureProvider struct {
    *provider.BaseProvider
    deployment string
    apiVersion string
    // ...
}
```

When to use embedding:

- Multiple sibling implementations need the same fields and method bodies.
- The shared state is simple (a few fields, no configuration).
- The concrete type genuinely **is-a** instance of the base type.

When **not** to use embedding:

- Behavior varies per implementation — use composition with an interface
  field instead.
- You want to model an inheritance tree — Go is not that language.
- The shared portion is one method — just duplicate it; embedding adds
  conceptual weight that two lines of code do not justify.

## Semantic getters

Provide methods that extract meaningful values. Do not require callers to
traverse internal structure.

Bad — caller knows the internal layout and panics on empty slices:

```go
content := chunk.Choices[0].Delta.Content
```

Good — caller asks for what it wants:

```go
content := response.Text()
toolCalls := response.ToolCalls()
```

The `Text()` accessor handles the multi-block content shape and returns an
empty string when there's nothing extractable. The `ToolCalls()` accessor
filters content blocks to just the tool-use ones. Both isolate callers from
the structural decisions made inside `response`.

Why this matters:

- Internal layout can evolve (e.g., adding a new content block type) without
  breaking every caller.
- Bounds checks happen once, in the accessor, not at every call site.
- The accessor name documents the intent (`FirstTextContent`, `ToolCalls`)
  better than the field-traversal expression.

This is principle 4 (encapsulation over exposure) applied at the API surface.

## Cross-package boundaries

When a package imports another package, the import should be of an interface,
not a concrete type. If you need a concrete, the dependency should be
resolved at a composition root rather than threaded through business logic.

Example:

```go
// agent.New takes interfaces, not concretes
func New(cfg *config.AgentConfig, p provider.Provider, f format.Format) Agent
```

Both `provider.Provider` and `format.Format` are interfaces. The concretes
(`*azure.AzureProvider`, `*openai.Format`) are resolved in `cmd/server/`:

```go
// cmd/server/server.go (sketch)
provider, err := provider.Create(cfg.Provider) // returns provider.Provider
format, err := format.Create(cfg.Format)       // returns format.Format
agent := agent.New(cfg, provider, format)
```

See [constructors.md#composition-root](constructors.md#composition-root) for
where the composition root sits in each project type. See
[registry-factory.md](registry-factory.md) for how `provider.Create` resolves
the right concrete from a string name.

Rule of thumb: business-logic packages import interfaces; the `cmd/`
composition root imports concretes.
