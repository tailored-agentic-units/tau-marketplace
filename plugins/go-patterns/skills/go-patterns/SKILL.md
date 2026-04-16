---
name: go-patterns
description: >
  Go design patterns, project layout, error handling, configuration lifecycle,
  CI/CD release workflows, and multi-module workspace coordination for the tau
  ecosystem. Apply this skill whenever you design a new Go package, define an
  interface, structure a config file, choose a constructor signature, write a
  test, set up release automation, or organize a workspace of related modules.
  Triggers: interface design, error handling, package structure, dependency
  hierarchy, configuration lifecycle, parameter encapsulation, Go idioms,
  black-box testing, registry/factory pattern, release workflow, CHANGELOG,
  multi-module workspace, go.work, doc.go, .claude settings, Makefile.
---

# Go Patterns

Architectural conventions for building Go libraries, CLIs, and web services in
the tau ecosystem. Patterns are derived from real projects: `tau/protocol`,
`tau/format`, `tau/provider`, `tau/agent`, `tau/orchestrate`, `agent-lab`,
`herald`, `go-agents`, and `go-agents-orchestration`.

## When This Skill Applies

This skill covers three Go project archetypes:

- **Library** — published Go modules consumed by other repos. Examples:
  `tau/protocol`, `tau/agent`, `go-agents`.
- **CLI / web service** — long-running binaries with subcommands. Examples:
  `herald`, `agent-lab`.
- **Library workspace** — repos containing multiple related Go modules with
  shared workspace coordination. Examples: `tau/` (root `go.work`),
  `tau/format` + `tau/format/openai` + `tau/format/converse`.

This skill explicitly does **not** cover TUI applications. None of the
reference projects are TUIs and the patterns here have not been validated for
that shape.

## Core Principles

The eight principles below orient every other decision in this skill. Each
principle has a one-line statement, a canonical illustration, and a pointer to
the reference file that develops it in detail.

### 1. Configuration is ephemeral

Configuration exists only during initialization. Load it, transform it into
domain values, then discard it. Runtime structs hold values, not config.

```go
cfg, _ := config.Load()
db, _ := database.New(&cfg.Database, logger) // values extracted now
// cfg goes out of scope; runtime never references it again
```

See: [references/configuration.md](references/configuration.md).

### 2. Interface-based layer interconnection

Layers communicate through interfaces, not concrete types. The **consumer**
defines the interface; the **lower layer** provides implementations.

```go
// agent.go — agent (consumer) defines what it needs from a provider
func New(cfg *config.AgentConfig, p provider.Provider, f format.Format) Agent
```

See: [references/package-design.md](references/package-design.md).

### 3. Parameter encapsulation

More than two parameters? Use a struct. The struct documents what the
operation requires and serializes cleanly when needed.

```go
// Instead of: Execute(ctx, capability, input, timeout, retries, headers)
// Use:        Execute(ctx, request ExecuteRequest)
```

See: [references/constructors.md](references/constructors.md).

### 4. Encapsulation over exposure

Provide semantic accessors for nested or computed state. Do not require
callers to traverse internal structure.

```go
// Good — caller asks for what it wants
text := response.Text()

// Bad — caller knows the internal layout and panics on empty slices
text := response.Choices[0].Delta.Content
```

See: [references/package-design.md](references/package-design.md#semantic-getters).

### 5. Single responsibility per package

Each package has one clear purpose. `protocol/config` holds configuration
types. `protocol/response` holds response types. `agent/client` holds the
HTTP client. Names like `types`, `utils`, `common`, or `helpers` violate this
rule and should not exist.

See: [references/project-layout.md](references/project-layout.md).

### 6. Unidirectional dependencies

Lower layers do not import higher layers. Shared types belong in the lowest
layer that needs them. Use dependency inversion (interface ownership in the
consumer) when a lower layer needs behavior from a higher one.

```
protocol  ←  format, provider  ←  agent  ←  orchestrate
```

See: [references/package-design.md](references/package-design.md#dependency-direction).

### 7. Black-box testing

Tests live in `tests/` mirroring source structure, with `package <pkg>_test`.
Black-box tests verify the public contract; in-package tests are reserved for
unexported helpers that cannot be exercised through the public API.

```go
// tests/agent_test.go
package agent_test

import "github.com/tailored-agentic-units/agent"
```

See: [references/testing.md](references/testing.md).

### 8. Modern Go idioms

Use the language as it exists today. Baseline is Go 1.26.

- `wg.Go(func())` — Go 1.25 replaces Add/goroutine/Done
- `maps.Copy(dst, src)` — Go 1.21 for map merging
- `min(a, b)` / `max(a, b)` — Go 1.21 builtins
- `for range n` — Go 1.22 integer range
- `errors.Join` — Go 1.20, scoped to aggregation boundaries only
- `slices.Contains` etc. — Go 1.21 standard slice helpers

See: [references/modern-idioms.md](references/modern-idioms.md).

## Reference Index

Reference files load on demand. Consult only the files relevant to the task.

**Structure & Design**
- [project-layout.md](references/project-layout.md) — directory structures
  per project type; sub-package vs sub-module depth rules
- [package-design.md](references/package-design.md) — interface placement,
  dependency direction, naming, embedding, semantic getters
- [constructors.md](references/constructors.md) — `New()` shapes, struct
  configs, runtime options, composition root

**Runtime Behavior**
- [configuration.md](references/configuration.md) — three-phase loading,
  ephemeral lifecycle, `Default*Config`/`Merge`/`Finalize`
- [error-handling.md](references/error-handling.md) — sentinels, typed
  errors with `Unwrap`, retry classification, `errors.Join` boundaries
- [concurrency.md](references/concurrency.md) — `sync.WaitGroup.Go`,
  context propagation, lifecycle Coordinator, streaming channels

**Code Style**
- [modern-idioms.md](references/modern-idioms.md) — Go 1.21–1.26 features
- [testing.md](references/testing.md) — black-box layout, table-driven,
  `httptest.NewServer`, mock packages, atomic assertions

**Patterns**
- [registry-factory.md](references/registry-factory.md) — pluggable
  implementations registered by name
- [workspace-multimodule.md](references/workspace-multimodule.md) —
  `go.work`, sub-modules, replace directives, version coordination

**Project Infrastructure**
- [ci-cd-release.md](references/ci-cd-release.md) — release workflow
  templates, tag conventions, CHANGELOG format
- [tooling-claude.md](references/tooling-claude.md) — Makefile,
  `.claude/settings.json`, `CLAUDE.md`, `doc.go`

## Quick Decision Table

| If you are... | Read... |
|---|---|
| Creating a new package | [project-layout.md](references/project-layout.md), [package-design.md](references/package-design.md) |
| Defining an error type | [error-handling.md](references/error-handling.md) |
| Designing a config struct | [configuration.md](references/configuration.md) |
| Writing a `New()` constructor | [constructors.md](references/constructors.md) |
| Adding a test file | [testing.md](references/testing.md) |
| Spawning a goroutine | [concurrency.md](references/concurrency.md) |
| Building a plugin/extension system | [registry-factory.md](references/registry-factory.md) |
| Splitting a repo into multiple modules | [workspace-multimodule.md](references/workspace-multimodule.md) |
| Setting up a release workflow | [ci-cd-release.md](references/ci-cd-release.md) |
| Writing a Makefile or `.claude/settings.json` | [tooling-claude.md](references/tooling-claude.md) |
| Reaching for a Go 1.20+ feature | [modern-idioms.md](references/modern-idioms.md) |
