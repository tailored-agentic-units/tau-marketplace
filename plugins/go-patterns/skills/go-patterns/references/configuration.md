# Configuration

Configuration is loaded once at startup, transformed into domain values, then
discarded. This file covers the two configuration styles (library vs
application), the three-phase loading pattern for applications, and the
factory/merge/finalize trio that supports both.

## Contents

1. [Two styles: library vs application](#two-styles-library-vs-application)
2. [Library style — struct-only](#library-style--struct-only)
3. [Application style — three-phase loading](#application-style--three-phase-loading)
4. [`Default*Config` factories](#defaultconfig-factories)
5. [`Merge` pattern](#merge-pattern)
6. [`Finalize` / `validate`](#finalize--validate)
7. [Configuration is ephemeral](#configuration-is-ephemeral)
8. [Environment variables](#environment-variables)
9. [Where config types live](#where-config-types-live)

## Two styles: library vs application

| Style | Project type | Loading | Constructor |
|---|---|---|---|
| Struct-only | library | Caller's responsibility | Caller assembles struct |
| Three-phase JSON | application (CLI/service) | Internal `config.Load()` | App owns the loader |

Libraries do not prescribe a loading mechanism. They expose a config struct,
a factory for defaults, and a merge method. The caller decides whether to
populate it from JSON, YAML, env vars, command-line flags, or in-memory
construction.

Applications own their loader. The loader knows where the config file lives,
what overlays are applied, what environment variables override what fields,
and when to validate.

## Library style — struct-only

A library exposes a config struct with JSON tags, a `Default*Config()`
factory, and a `Merge` method:

```go
type AgentConfig struct {
    Name         string          `json:"name"`
    SystemPrompt string          `json:"system_prompt,omitempty"`
    Format       string          `json:"format,omitempty"`
    Client       *ClientConfig   `json:"client,omitempty"`
    Provider     *ProviderConfig `json:"provider"`
    Model        *ModelConfig    `json:"model"`
}

func DefaultAgentConfig() AgentConfig {
    return AgentConfig{
        Name:         "default-agent",
        SystemPrompt: "",
        Format:       "openai",
        Client:       DefaultClientConfig(),
        Provider:     DefaultProviderConfig(),
        Model:        DefaultModelConfig(),
    }
}
```

Source: `/home/jaime/tau/protocol/config/agent.go:9-31`

The library may also provide a convenience loader, but consumers are not
required to use it:

```go
func LoadAgentConfig(filename string) (*AgentConfig, error) {
    config := DefaultAgentConfig()

    data, err := os.ReadFile(filename)
    if err != nil {
        return nil, fmt.Errorf("failed to read config file: %w", err)
    }

    var loaded AgentConfig
    if err := json.Unmarshal(data, &loaded); err != nil {
        return nil, fmt.Errorf("failed to parse config file: %w", err)
    }

    config.Merge(&loaded)
    return &config, nil
}
```

Source: `/home/jaime/tau/protocol/config/agent.go:73-91`

A library consumer can either call `LoadAgentConfig("config.json")` or
build the struct directly:

```go
cfg := config.DefaultAgentConfig()
cfg.Provider.Name = "ollama"
cfg.Model.Name = "llama3"
```

## Application style — three-phase loading

An application's `config.Load()` follows three phases in order:

1. **Read base + overlays.** Start with `config.json`, then apply
   `config.${HERALD_ENV}.json` if it exists, then apply `secrets.json` if it
   exists. Each overlay is merged into the previous result.
2. **Apply environment variables.** Each subsystem declares an `Env` struct
   listing its environment variable names. `loadEnv()` walks them and
   overrides matching fields when the env var is non-empty.
3. **Finalize.** Validate parsed values, parse durations, normalize URLs,
   delegate to per-subsystem `Finalize()` methods that combine load + env
   reading + validation in one call.

```go
func Load() (*Config, error) {
    cfg := &Config{}

    if _, err := os.Stat(BaseConfigFile); err == nil {
        loaded, err := load(BaseConfigFile)
        if err != nil {
            return nil, err
        }
        cfg = loaded
    }

    if path := overlayPath(); path != "" {
        overlay, err := load(path)
        if err != nil {
            return nil, fmt.Errorf("load overlay %s: %w", path, err)
        }
        cfg.Merge(overlay)
    }

    if _, err := os.Stat(SecretsConfigFile); err == nil {
        secrets, err := load(SecretsConfigFile)
        if err != nil {
            return nil, err
        }
        cfg.Merge(secrets)
    }

    if err := cfg.finalize(); err != nil {
        return nil, fmt.Errorf("finalize config: %w", err)
    }

    return cfg, nil
}
```

Source: `/home/jaime/code/herald/internal/config/config.go:89-121`

The `finalize()` method composes per-subsystem `Finalize` calls:

```go
func (c *Config) finalize() error {
    c.loadDefaults()
    c.loadEnv()

    if err := c.validate(); err != nil {
        return err
    }
    if err := FinalizeAgent(&c.Agent); err != nil {
        return fmt.Errorf("agent: %w", err)
    }
    if err := c.Auth.Finalize(authEnv); err != nil {
        return fmt.Errorf("auth: %w", err)
    }
    if err := c.Database.Finalize(databaseEnv); err != nil {
        return fmt.Errorf("database: %w", err)
    }
    // ...
    return nil
}
```

Source: `/home/jaime/code/herald/internal/config/config.go:139-165`

Why three phases:

- Overlays let environments customize without duplicating the entire base
  config.
- Environment variables let containers and orchestrators override individual
  values without rewriting files.
- Finalization is the single place where validation lives; it runs after
  every source has had its say.

## `Default*Config` factories

Every config type has a `Default*Config()` function that returns a value with
sensible non-empty defaults. Defaults make the config struct usable without
any input — useful for libraries, tests, and as a starting point for merges.

```go
func DefaultAgentConfig() AgentConfig {
    return AgentConfig{
        Name:         "default-agent",
        Format:       "openai",
        Client:       DefaultClientConfig(),  // delegate to sub-defaults
        Provider:     DefaultProviderConfig(),
        Model:        DefaultModelConfig(),
    }
}
```

Conventions:

- Returns a value, not a pointer (caller gets a copy they can freely modify).
- Sub-config defaults delegate to their own `Default*Config()` functions.
- This mirrors Go's zero-value idiom while making non-zero defaults explicit
  and testable.

## `Merge` pattern

`Merge(source)` overlays non-empty fields from `source` onto the receiver:

```go
func (c *AgentConfig) Merge(source *AgentConfig) {
    if source.Name != "" {
        c.Name = source.Name
    }
    if source.SystemPrompt != "" {
        c.SystemPrompt = source.SystemPrompt
    }
    if source.Format != "" {
        c.Format = source.Format
    }
    if source.Client != nil {
        if c.Client == nil {
            c.Client = source.Client
        } else {
            c.Client.Merge(source.Client)
        }
    }
    // Provider, Model: same pattern
}
```

Source: `/home/jaime/tau/protocol/config/agent.go:33-71`

Rules:

- **Non-empty source wins.** Empty strings, nil pointers, and zero values do
  not override existing values.
- **Sub-structs delegate.** `Client.Merge` knows how to merge clients; parent
  doesn't need to know client internals.
- **Nil handling**: if the receiver's pointer field is nil, take the source's
  value wholesale; otherwise merge into the existing one.
- **Don't use reflection.** Explicit field handling reads better and catches
  type mismatches at compile time.

## `Finalize` / `validate`

Validation does not run at load time. It runs once, **after** all sources
have been merged and **before** any consumer reads the config.

```go
func (c *DatabaseConfig) Finalize(env *Env) error {
    c.loadDefaults()
    c.loadEnv(env)
    return c.validate()
}
```

This pattern lets each subsystem own its three steps (defaults, env, validate)
behind a single `Finalize(env)` method called from the parent's `finalize()`.

Why deferred validation matters:

- A required field can come from any source — defaults, file, overlay,
  secrets, or env var. Validating after a load step rejects values that the
  next source would have provided.
- Duration strings ("30s") and other parsed values need to be valid only at
  runtime use, not at every intermediate merge step.
- Cross-field validation (e.g., "if X is set, Y must be set") only makes
  sense once everything is in place.

## Configuration is ephemeral

After `config.Load()` and construction, the config struct should be
discarded. Subsystems hold the **values** they need, not a reference to the
config.

Correct:

```go
cfg, _ := config.Load()
db, _ := database.New(&cfg.Database, logger) // reads connection string now
storage, _ := storage.New(&cfg.Storage, logger)
// cfg can go out of scope; subsystems retain only what they need
```

Incorrect:

```go
type Service struct {
    cfg *Config // BAD — config leaks into runtime
}

func (s *Service) Connect() error {
    return connect(s.cfg.Database.URL) // re-reading at runtime
}
```

Why this matters:

- Configuration represents **initialization intent**. Runtime state must not
  query it; runtime state operates on already-transformed values.
- Holding config references creates uncertainty about when settings are read
  (at startup vs at first use vs on every call).
- Tests can construct subsystems with synthetic values without building a
  whole `Config` graph.

The exception — long-running config watchers that hot-reload a file — is a
separate concern, not a config pattern. If you build one, design it as its
own subsystem with its own contract.

## Environment variables

Environment variables are the third source in the load chain. To keep their
handling structured:

- Declare env var names as exported string constants at package scope.
- Group related env vars into an `Env` struct passed to the subsystem's
  `Finalize(env)` method.

```go
const (
    EnvHeraldEnv             = "HERALD_ENV"
    EnvHeraldShutdownTimeout = "HERALD_SHUTDOWN_TIMEOUT"
    EnvHeraldVersion         = "HERALD_VERSION"
)

var databaseEnv = &database.Env{
    Host:            "HERALD_DB_HOST",
    Port:            "HERALD_DB_PORT",
    Name:            "HERALD_DB_NAME",
    User:            "HERALD_DB_USER",
    Password:        "HERALD_DB_PASSWORD",
    SSLMode:         "HERALD_DB_SSL_MODE",
    MaxOpenConns:    "HERALD_DB_MAX_OPEN_CONNS",
    MaxIdleConns:    "HERALD_DB_MAX_IDLE_CONNS",
    ConnMaxLifetime: "HERALD_DB_CONN_MAX_LIFETIME",
    ConnTimeout:     "HERALD_DB_CONN_TIMEOUT",
}
```

Source: `/home/jaime/code/herald/internal/config/config.go:16-51`

Why a struct, not direct calls to `os.Getenv` inside the subsystem:

- The subsystem stays portable — it doesn't bake in `HERALD_*` naming.
- Other applications can reuse the subsystem with their own env var prefix.
- Tests can pass a stub `Env` struct that points at temporary names.

## Where config types live

| Where | What | Why |
|---|---|---|
| `protocol/config/` | Foundational types (`AgentConfig`, `ClientConfig`, etc.) | Every higher layer reads them; lowest-layer ownership prevents import cycles. |
| `internal/config/` | App-specific composition (`Config` struct that embeds foundational types) | App owns its loader; types are private to the app. |
| `pkg/<subsystem>/` | Subsystem-specific config (`database.Config`, `storage.Config`) | Each `pkg/` library defines its own config + `Default()` + `Merge()` + `Finalize(env)`. |

The application's `Config` composes everything:

```go
type Config struct {
    Agent           gaconfig.AgentConfig `json:"agent"`     // foundational
    Auth            auth.Config          `json:"auth"`      // pkg/auth
    Database        database.Config      `json:"database"`  // pkg/database
    Storage         storage.Config       `json:"storage"`   // pkg/storage
    Server          ServerConfig         `json:"server"`    // app-defined
    API             APIConfig            `json:"api"`       // app-defined
    ShutdownTimeout string               `json:"shutdown_timeout"`
    Version         string               `json:"version"`
}
```

Source: `/home/jaime/code/herald/internal/config/config.go:60-70`

This is the composition root for configuration: the one place where every
layer's config types come together and become a single loadable object.
