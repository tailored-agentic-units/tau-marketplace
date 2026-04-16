# Tooling and `.claude/` Setup

The canonical Makefile target set, the shape of `.claude/settings.json`,
the two CLAUDE.md modes (mentorship vs lifecycle), and the `doc.go`
conventions that produce good godoc output.

## Contents

1. [Makefile](#makefile)
2. [`.claude/settings.json`](#claudesettingsjson)
3. [`CLAUDE.md` conventions](#claudemd-conventions)
4. [`doc.go` conventions](#docgo-conventions)

## Makefile

The canonical Makefile structure derived from agent-lab and herald:

```makefile
.PHONY: dev build web run test vet clean

# Development: build web assets and run server
dev: web run

# Production build
build: web
	go build -o bin/server ./cmd/server

# Build web assets (omit if no frontend)
web:
	cd web && bun install && bun run build

# Run the server (or main binary)
run:
	go run ./cmd/server/

# Run tests against the tests/ directory (black-box)
test:
	go test ./tests/...

# Static analysis
vet:
	go vet ./...

# Clean build artifacts
clean:
	rm -rf bin/
	rm -rf web/app/dist
```

Source: `/home/jaime/code/agent-lab/Makefile`

Target matrix by project type:

| Target | Library | CLI | Service |
|---|---|---|---|
| `test` | yes | yes | yes |
| `vet` | yes | yes | yes |
| `build` | optional | yes | yes |
| `run` | no | yes | yes |
| `dev` | no | maybe | yes |
| `web` | no | no | only if frontend |

Add the release-tag targets from
[ci-cd-release.md#makefile-integration](ci-cd-release.md#makefile-integration):

```makefile
.PHONY: tag tag-module

tag:
	git tag -a $(VERSION) -m "Release $(VERSION)"
	git push origin $(VERSION)

tag-module:
	git tag -a $(MODULE)/$(VERSION) -m "Release $(MODULE) $(VERSION)"
	git push origin $(MODULE)/$(VERSION)
```

Conventions:

- **`.PHONY` at the top** listing every non-file target.
- **One-line comment per target.**
- **Tests target `./tests/...`** — black-box-only, mirrors source structure
  (see [testing.md](testing.md)).
- **Don't combine `go test` with linting in one target.** They report
  different things; CI users want to see them separately.

## `.claude/settings.json`

The canonical shape with explicit Bash and Skill allow-listing:

```json
{
  "plansDirectory": "./.claude/plans",
  "permissions": {
    "allow": [
      "Bash(gh auth status*)",
      "Bash(gh issue list*)",
      "Bash(gh issue view*)",
      "Bash(gh project field-list*)",
      "Bash(gh project item-list*)",
      "Bash(gh project list*)",
      "Bash(gh project view*)",
      "Bash(go build*)",
      "Bash(go mod tidy*)",
      "Bash(go run*)",
      "Bash(go test*)",
      "Bash(go vet*)",
      "Bash(grep:*)",
      "Bash(ls:*)",
      "Skill(dev-workflow:dev-workflow)",
      "Skill(github-cli:github-cli)",
      "Skill(go-patterns:go-patterns)",
      "Skill(project-management:project-management)",
      "Skill(skill-creator:skill-creator)",
      "WebSearch"
    ],
    "deny": ["Read(secrets.json)"]
  }
}
```

Source: `/home/jaime/code/herald/.claude/settings.json`

Conventions:

- **`plansDirectory: "./.claude/plans"`** — relative to repo root, scopes
  plan files alongside the rest of the `.claude/` setup.
- **Allow-list explicit Bash patterns**: `gh` for GitHub operations, `go` for
  build and test, `grep`/`ls` for navigation. Include only what the
  project actually needs.
- **Allow Skills the project depends on**: at minimum `go-patterns` and
  `skill-creator`; for service-shaped projects, add `dev-workflow`,
  `project-management`, and `github-cli`.
- **Deny secrets**: `Read(secrets.json)` blocks accidental ingestion of
  credentials.
- **`settings.json` is committed**; **`settings.local.json` is gitignored**
  for per-developer overrides.

Allow-list by project type:

- **Library**: `go-patterns`, `skill-creator`. Bash for `go test`, `go vet`,
  `go mod tidy`, `gh` for issue/PR work.
- **CLI / service**: add `dev-workflow`, `project-management`, plus any
  domain-specific custom skills.

## `CLAUDE.md` conventions

Two modes are observed in the reference projects.

### Mentorship mode

```markdown
You are an expert in the following areas of expertise:

- Building libraries, tools, and services with the Go programming language
- Building agentic workflows and tooling using LLM platforms
- LLM provider APIs and integration patterns
- Multi-agent coordination architectures and protocol standards

Whenever I reach out to you for assistance, I'm not asking you to make
modifications to my project; I'm merely asking for advice and mentorship.
```

Source: `/home/jaime/code/go-agents/CLAUDE.md` (excerpt)

Use mentorship mode for:

- Solo learning projects.
- Architectural exploration where the user wants pushback before
  implementation.
- Libraries where the user retains explicit ownership of code decisions.

### Lifecycle mode

```markdown
# Herald

Go web service for classifying DoD PDF documents' security markings...

## Architecture

Herald follows the Layered Composition Architecture (LCA): cold start
(config load, subsystem creation) → hot start (connections, HTTP listen) →
graceful shutdown (reverse-order teardown).

### Configuration Pattern

Every config struct follows the three-phase finalize pattern:
1. `loadDefaults()` — hardcoded fallbacks
2. `loadEnv(env)` — environment variable overrides
3. `validate()` — validate final values

## AI Responsibilities

### Testing

All test authorship is an AI responsibility. Tests live in `tests/`
mirroring the source structure. Black-box only (`package <name>_test`).
```

Source: `/home/jaime/code/herald/.claude/CLAUDE.md` (excerpt)

Use lifecycle mode for:

- Mature projects with multiple contributors (human or AI).
- Production services where workflow conventions matter as much as code
  conventions.
- Repos where AI takes ownership of specific responsibilities (testing,
  godoc, API documentation).

### Common structure

Both modes typically include:

```markdown
# <Project Name>

## Purpose / Description

## Workflow / Mentorship Relationship

## Key Conventions
- Patterns to follow (link to relevant skills)
- Tooling expectations

## Commands / Skills
- List of skills the AI is expected to invoke

## Out of Scope
```

Note the division of labor: **the go-patterns skill provides Go patterns**;
**CLAUDE.md provides project-specific conventions** (release cadence,
architectural decisions specific to this codebase, ownership boundaries).
Don't duplicate skill content in CLAUDE.md.

## `doc.go` conventions

Each package has exactly one `doc.go` containing only the package comment.

### Top-level structure

```go
// Package agent provides a high-level interface for LLM interactions.
// It wraps the client layer with convenient methods for common operations
// like chat, vision, tools, and embeddings.
//
// # Agent Interface
//
// The Agent interface provides protocol-specific methods that accept
// pre-built message slices, allowing callers to manage system prompts
// and conversation history externally:
//
//	type Agent interface {
//	    ID() string
//	    Client() client.Client
//	    Provider() provider.Provider
//	    Format() format.Format
//	    Model() *model.Model
//
//	    Chat(ctx, messages, opts...) (*response.Response, error)
//	    ChatStream(ctx, messages, opts...) (<-chan *response.StreamingResponse, error)
//	    // ...
//	}
//
// # Creating an Agent
//
// Agents are created via explicit dependency injection — the caller provides
// the provider and format instances:
//
//	p, _ := provider.Create(cfg.Provider)
//	f, _ := format.Create(cfg.Format)
//	a := agent.New(cfg, p, f)
//
// # Options Management
//
// All protocol methods accept optional parameters that are merged with
// model defaults. Request options take precedence over model defaults.
//
// # Error Handling
//
// The package provides AgentError with categorization (init, llm),
// unique identification, and contextual metadata for structured error reporting.
//
// # Thread Safety
//
// Agents are safe for concurrent use. Multiple goroutines can call protocol
// methods simultaneously on the same agent instance.
package agent
```

Source: `/home/jaime/tau/agent/doc.go`

Structural conventions:

- **Opening paragraph** in 2–3 sentences stating the package's purpose.
- **`# Section` headers** render as `##` headings in godoc.
- **Tab-indented code blocks** for examples (godoc renders them as code).
- **Concurrency section** for any package with exported goroutine-safe
  types.
- **Error handling section** for any package with typed errors.

### Minimal form

For sub-packages, a brief description with sub-package listing suffices:

```go
// Package protocol provides the foundation types for LLM interaction in the TAU ecosystem.
//
// It defines the Protocol type representing different LLM capabilities, the Message type
// for conversation structures, and supporting types for configuration, response handling,
// model runtime, and streaming.
//
// Sub-packages:
//   - config: Agent, client, provider, and model configuration with JSON serialization
//   - response: Unified response types with content blocks (text, tool use)
//   - model: Runtime model type bridging configuration to domain
//   - streaming: StreamReader interface for streamed LLM responses
package protocol
```

Source: `/home/jaime/tau/protocol/doc.go`

### Where it goes

- **One `doc.go` per package**, no exceptions.
- **Contains only the package comment.** No types, no vars, no functions.
- **Place at the top of the package directory** alphabetically (so it
  appears first in `ls`).
