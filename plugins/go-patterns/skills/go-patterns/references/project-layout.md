# Project Layout

Directory conventions for the three Go project archetypes (library,
CLI/service, library workspace), plus universal rules that apply to all of
them: where `doc.go` lives, how `tests/` mirrors source, what `_project/` and
`_context/` are for, and the depth rule that distinguishes sub-packages from
sub-modules.

## Contents

1. [Project shape selector](#project-shape-selector)
2. [Library layout](#library-layout) — `tau/protocol` style
3. [CLI / service layout](#cli--service-layout) — `herald` style
4. [Single-binary application](#single-binary-application)
5. [Universal conventions](#universal-conventions)
6. [Depth nuance: sub-packages vs sub-modules](#depth-nuance-sub-packages-vs-sub-modules)
7. [Anti-patterns](#anti-patterns)

## Project shape selector

| Project goal | Shape | Reference projects |
|---|---|---|
| Published library imported by other repos | Library | `tau/protocol`, `tau/agent`, `go-agents` |
| Long-running service / CLI binary | CLI / service | `herald`, `agent-lab` |
| Multiple related modules in one repo | Library workspace | `tau/`, `tau/format`, `tau/provider` |

Workspace shape is orthogonal — a library workspace contains library shapes
inside it. See [workspace-multimodule.md](workspace-multimodule.md).

## Library layout

A library exposes its API at the package root. There is no `cmd/`, no
`internal/`, no `pkg/`. Sub-packages exist only when there is a cohesive
sub-domain (e.g., `protocol/config`, `protocol/response`).

```
protocol/
├── CHANGELOG.md
├── README.md
├── doc.go
├── go.mod
├── protocol.go        # root-package API: Protocol type + constants
├── message.go         # root-package types: Message, Role, ToolCall
├── config/            # sub-package: configuration types
│   ├── doc.go
│   ├── agent.go
│   ├── client.go
│   ├── model.go
│   └── provider.go
├── model/             # sub-package: runtime model type
├── response/          # sub-package: response + content blocks
├── streaming/         # sub-package: StreamReader interface
├── tests/             # black-box tests, mirrors source
│   ├── protocol_test.go
│   ├── config/
│   ├── response/
│   └── streaming/
└── _project/
    └── README.md
```

Source: `/home/jaime/tau/protocol/`

Rules for libraries:

- Root-level `*.go` files form the main package API.
- A new file is justified when its types serve a different concern, not
  when an existing file gets long.
- Sub-packages only when there is a cohesive sub-domain that merits its
  own import path.
- `tests/` mirrors source 1:1; one source file produces one test file.

## CLI / service layout

CLI and service projects share a layered structure with explicit boundaries
between entry points (`cmd/`), private domain code (`internal/`), and reusable
libraries that could in principle be extracted (`pkg/`).

```
herald/
├── CHANGELOG.md
├── Makefile
├── go.mod
├── config.json              # base configuration (committed)
├── config.docker.json       # environment overlay
├── secrets.json             # gitignored
├── cmd/
│   ├── migrate/
│   │   └── main.go
│   └── server/
│       ├── main.go          # signal handling, lifecycle wiring
│       ├── server.go        # NewServer(cfg) — composition root
│       ├── http.go
│       └── modules.go
├── internal/                # app-private; cannot be imported externally
│   ├── api/
│   ├── classifications/
│   ├── config/              # 3-phase Load() lives here
│   ├── documents/
│   ├── infrastructure/      # shared init: db, storage, logger
│   ├── prompts/
│   └── workflow/
├── pkg/                     # reusable; importable by other repos
│   ├── auth/
│   ├── database/
│   ├── lifecycle/
│   ├── middleware/
│   ├── repository/
│   └── storage/
├── tests/                   # black-box tests, mirrors internal/ + pkg/
├── web/                     # frontend assets, if applicable
├── deploy/                  # terraform, container manifests
└── _project/                # planning docs, concept artifacts
```

Source: `/home/jaime/code/herald/`

Rules for CLI / service:

- Each binary is one directory under `cmd/<name>/`. The composition root
  (the function that wires concrete dependencies together) lives there.
- Anything under `internal/` is invisible to other modules; use this for
  domain logic that should never leak.
- `pkg/` is the seam where your code becomes shareable. Treat its
  exported API as a contract.
- Domain modules under `internal/` (e.g., `documents/`, `classifications/`)
  each own their package; cross-domain interop happens through interfaces.

## Single-binary application

Pure single-binary applications without subcommands are rare in this
ecosystem and typically only appear in throwaway tools. Even projects that
ship one primary binary tend to grow secondary tools (`cmd/migrate`,
`cmd/seed`, etc.). When in doubt, start with `cmd/<primary>/main.go` rather
than a root-level `main.go`. Adding `cmd/migrate/` later is trivial; moving
a root `main.go` into `cmd/server/` later is churn.

## Universal conventions

### `doc.go` placement

Every package has exactly one `doc.go` containing only the package comment.
No types, vars, or functions live in `doc.go`. The structured form uses
`# Section` headers (rendered as godoc subheadings):

```go
// Package agent provides a high-level interface for LLM interactions.
//
// # Agent Interface
//
// The Agent interface provides protocol-specific methods that accept
// pre-built message slices...
//
// # Creating an Agent
//
// Agents are created via explicit dependency injection:
//
//	p, _ := provider.Create(cfg.Provider)
//	f, _ := format.Create(cfg.Format)
//	a := agent.New(cfg, p, f)
//
// # Thread Safety
//
// Agents are safe for concurrent use...
package agent
```

Source: `/home/jaime/tau/agent/doc.go`

For a minimal sub-package listing, see `/home/jaime/tau/protocol/doc.go`.

### `tests/` directory mirroring

Tests live under `tests/`, not alongside source. The directory tree under
`tests/` matches the source tree exactly:

```
agent/
├── agent.go               →  tests/agent_test.go
├── client/client.go       →  tests/client/client_test.go
├── mock/agent.go          →  tests/mock/agent_test.go
└── tests/
    ├── agent_test.go
    ├── client/
    └── mock/
```

Test packages use the `_test` suffix (`package agent_test`) for black-box
testing. See [testing.md](testing.md) for details.

### Underscore-prefix directories

The leading underscore is a Go tooling convention for "ignore this directory":
`go build ./...` skips it, `go test ./...` skips it, IDEs typically dim it.

- **`_project/`** (tau convention) — project-level planning, concept docs,
  session artifacts. Examples: `/home/jaime/tau/agent/_project/`,
  `/home/jaime/code/herald/_project/`.
- **`_context/`** — development session history, prompts, archived guides.
  Example: `/home/jaime/code/agent-lab/_context/`.

These directories are documentation, not code. Don't import from them.

## Depth nuance: sub-packages vs sub-modules

The "max one level of sub-packages" rule is about **sub-packages within a
single Go module**. It does **not** apply to sub-modules — directories with
their own `go.mod` are independent modules that happen to be co-located, and
each one resets its own depth budget.

### Within a single module: max one level

Inside one module, do not nest sub-packages beyond one level:

```
# OK
herald/pkg/repository/

# NOT OK — two levels of sub-packages within the same module
herald/pkg/repository/users/queries/
```

If you find yourself wanting `pkg/repository/users/queries/`, flatten by
discriminating with filenames (`queries.go`, `users.go`) at the
`pkg/repository/` level, or extract `users/` as its own top-level package.

### Sub-modules can nest as needed

A directory with its own `go.mod` is a separate module. Sub-modules are how
you escape the single-module depth budget for related-but-separate concerns:

```
# All sub-modules — each has its own go.mod
tau/provider/azure/
tau/provider/bedrock/
tau/provider/ollama/
tau/format/openai/
tau/format/converse/
tau/container/docker/
```

Inside `tau/provider/azure/`, the same single-module rule applies again — the
sub-module is its own module, so the depth budget restarts at zero.

### When to extract a sub-module

Justifiable reasons to give a directory its own `go.mod`:

- It needs to be **released independently** from its parent.
- It has a **distinct dependency surface** (e.g., `provider/azure` pulls in
  Azure SDK; consumers of `provider/bedrock` should not have to download it).
- It is part of a **plugin-style architecture** where consumers select which
  implementations to depend on.
- It is **meaningfully consumable in isolation** — i.e., a real project would
  import only this module without the parent.

Splitting a directory into a sub-module purely to dodge the depth rule is the
wrong reason. Sub-modules add real coordination cost (separate `go.mod`,
separate CHANGELOG, separate tag).

See [workspace-multimodule.md](workspace-multimodule.md#sub-module-decomposition)
for the full sub-module decomposition guide.

## Anti-patterns

- **Generic-name packages**: `types`, `utils`, `common`, `helpers`, `shared`.
  Each one becomes a junk drawer. Name packages by what they provide.
- **Deep sub-package nesting** within a single module (see depth rule above).
- **Mixing `cmd/` and root-level `main.go`** in the same repo.
- **`internal/types/` as a dumping ground** for cross-domain types — those
  belong in the lowest layer that needs them. See
  [package-design.md](package-design.md#dependency-direction).
- **`pkg/` containing domain logic** rather than reusable libraries — domain
  logic belongs in `internal/`.
- **Empty packages** with one file containing one type — fold into a sibling
  unless the package boundary is genuinely meaningful.
