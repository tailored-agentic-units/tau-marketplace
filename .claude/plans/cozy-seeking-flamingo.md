# Plan: Rebuild go-patterns Skill

## Context

The current `go-patterns` skill at `plugins/go-patterns/skills/go-patterns/SKILL.md` is a single 78-line file that lists eight principles with one-paragraph explanations and a few inline examples. It's "woefully under-developed" relative to the patterns established across nine real Go projects (`~/code/go-agents`, `~/code/go-agents-orchestration`, `~/code/agent-lab`, `~/code/herald`, `~/tau/protocol`, `~/tau/orchestrate`, `~/tau/format`, `~/tau/provider`, `~/tau/agent`).

**Goal**: Expand `go-patterns` into a multi-file skill that captures the full architectural vocabulary used to build libraries, CLIs, and web services in this ecosystem, with progressive disclosure so SKILL.md stays scannable while detail lives in topic-specific reference files.

**User decisions** (already confirmed):
- **Config canonical**: JSON for applications (3-phase: defaults → file overlay → env vars → finalize); struct-only for libraries.
- **errors.Join**: Avoid in core; use only at aggregation boundaries (batch processors, validators, multi-resource shutdown).
- **Scope**: library, CLI, web service patterns only — explicitly NOT TUI (none demonstrated in source projects).
- **Organization**: SKILL.md (principles + index) + `references/` split by topic.

**Plugin version bump**: `0.1.0` → `0.2.0` (minor — net new content, no breaking changes to skill interface).

## Final Skill Structure

```
plugins/go-patterns/skills/go-patterns/
├── SKILL.md                         # ~230 lines: principles + index
└── references/
    ├── project-layout.md            # ~260 lines: dir structures by project type
    ├── package-design.md            # ~280 lines: interfaces, dependency direction
    ├── configuration.md             # ~290 lines: 3-phase loading, ephemeral
    ├── error-handling.md            # ~280 lines: typed errors, sentinels, retry
    ├── constructors.md              # ~230 lines: New(), struct configs, comp root
    ├── testing.md                   # ~260 lines: black-box, table-driven, mocks
    ├── concurrency.md               # ~250 lines: WaitGroup.Go, lifecycle, channels
    ├── modern-idioms.md             # ~220 lines: Go 1.22-1.26 features
    ├── registry-factory.md          # ~210 lines: plugin/extensibility pattern
    ├── ci-cd-release.md             # ~320 lines: workflows, tags, CHANGELOG
    ├── tooling-claude.md            # ~240 lines: Makefile, .claude, doc.go
    └── workspace-multimodule.md     # ~240 lines: go.work, submodules
```

Total ~3,310 lines. SKILL.md alone is loaded into context every invocation; reference files load only when their topic is consulted.

## Per-File Specification

Each reference file follows a uniform shape:
- Purpose paragraph at the top.
- Table of Contents if >150 lines.
- Sections with section-anchored cross-references (`See: references/<file>.md#<heading>`).
- Real code excerpts cited as `Source: <absolute-path>` immediately before the closing fence.
- Project-type tags (`[library]`, `[cli]`, `[service]`) on sections that differ across types; unmarked sections are universal.

### SKILL.md (~230 lines)

- **Frontmatter**: keep `name: go-patterns`; expand `description` triggers to add "CI/CD release workflows", "multi-module workspaces", "Go project layout", "registry pattern".
- **When This Skill Applies**: enumerate library/CLI/service; explicit non-goal of TUI.
- **Core Principles** (~130 lines, ~15 each): Configuration is ephemeral · Interface-based interconnection · Parameter encapsulation (>2 params → struct) · Encapsulation over exposure (semantic getters) · Single responsibility per package · Unidirectional dependencies · Black-box testing · Modern Go idioms. Each principle gets a 2-sentence statement + 1 canonical snippet + pointer to its reference file.
- **Reference Index** (~40 lines): grouped index of the 12 reference files (Structure & Design / Runtime Behavior / Code Style / Patterns / Project Infrastructure).
- **Quick Decision Table** (~25 lines): "If you are designing X → read Y" mapping for new packages, error types, configs, tests, goroutines, plugin systems, multi-module repos, release workflows.

### references/project-layout.md

Three project shapes: library (`tau/protocol` style — root packages, no `internal/`/`cmd/`/`pkg/`), CLI/service with subcommands (`herald` style — `cmd/{server,migrate}`, `internal/`, `pkg/`, `tests/`, `web/`), single-entry app (rare; pointer to prefer subcommand shape). Universal conventions: `doc.go` placement, `tests/` mirroring source 1:1, `_project/` and `_context/` underscore-prefix conventions. Subdirectory depth rule (see *Depth nuance* below). Anti-patterns: util/common/helpers packages, deep nesting *within a single module*.

**Depth nuance** (own section): the "max 1 level of sub-packages" rule applies *within a single Go module*. **Sub-modules** (directories with their own `go.mod`) can nest as deep as the repo's logical decomposition requires, because each is an independent Go module that happens to be co-located. Examples:

- `tau/provider/azure/` — sub-module with own `go.mod`, fine.
- `tau/provider/bedrock/`, `tau/provider/ollama/` — sub-modules.
- `tau/container/docker/` — sub-module.
- `tau/format/openai/`, `tau/format/converse/` — sub-modules.

What the rule *does* prohibit: nesting *sub-packages* within a single module beyond one level (`pkg/repository/users/queries/` is not OK because `pkg/repository/users/` and `pkg/repository/users/queries/` are sub-packages within one module). Within a sub-module like `tau/provider/azure/`, the same rule applies *internally* (the sub-module is its own module so the depth budget restarts).

Show this with a small tree contrasting `tau/provider/` (sub-modules nested deeply) vs `herald/pkg/` (sub-packages flat).

### references/package-design.md

Package naming (singular lowercase, no `types`/`utils`). Dependency direction with the canonical tau import graph (`protocol ← format,provider ← agent ← orchestrate`). Interfaces owned by consumer (high-level package defines). Unexported structs returned as exported interfaces (`type agent struct` returned as `Agent`). `BaseProvider` embedding pattern from `~/tau/provider/base.go`. Semantic getters (Principle 4 expanded). Cross-package boundaries (import the interface, not the concrete; concretes resolved in composition root). Cross-references the `references/project-layout.md#depth-nuance` for sub-module vs sub-package distinction (sub-modules are how to extend a logical "package family" beyond the single-module depth budget).

### references/configuration.md

Two styles chart (library vs application). Library style: struct + `Default*Config()` factory + `Merge()` + optional `Load*Config(filename)` helper. Application 3-phase: defaults → `config.json` + `config.${ENV}.json` + `secrets.json` overlays → env vars → `finalize()` validates. `Default*Config` factory convention. Merge pattern (non-empty source wins, sub-structs delegate). Finalize/validate at instantiation only. Configuration is ephemeral (never store config in runtime structs). Environment variables: per-subsystem `Env` struct with constants. Where config types live: `protocol/config/` for foundational, `internal/config/` for app composition.

### references/error-handling.md

`errors.go` file convention. Sentinel errors (`Err` prefix, lowercase, no period). Typed errors with metadata + `Unwrap()` (full `AgentError` excerpt from `~/tau/agent/errors.go:30-91`). Functional options for error enrichment (`WithCode`, `WithCause`, `WithAgent`, `WithID`) — explicitly noted as the one place functional options belong. Wrapping with `%w`. Retry classification using `errors.Is`/`errors.As` (excerpt from `~/tau/agent/client/retry.go:40-78`). `HTTPStatusError` typed error pattern. `errors.Join` confined to aggregation boundaries with good/bad examples. Error message style (lowercase, no punctuation).

### references/constructors.md

`New()` returns interface or concrete (decision tree). Struct configs preferred over functional options for constructors (citing `agent.New(cfg, p, f)` shape). Variadic options only for builders/mocks (citing `mock.NewMockAgent(WithAgentID(...))`). Runtime options as `map[string]any` with `maps.Copy` merge (full `mergeOptions` excerpt). Composition root location: `cmd/<name>/` for apps, no composition root for libraries. Full `Infrastructure` excerpt from `~/code/agent-lab/internal/infrastructure/infrastructure.go`. Anti-patterns: panicking constructors, init-time I/O, 6-parameter constructors.

### references/testing.md

Black-box default (`package <pkg>_test` in `tests/<pkg>_test.go`); white-box only for unexported helpers. `tests/` mirrors source 1:1. Table-driven shape (anonymous struct, `name` first, `t.Run`, `want*` prefix). `httptest.NewServer` for real HTTP rather than mocking `http.Client`. Mock packages co-located (`agent/mock/`) with builder constructors. Concurrency assertions via `atomic.Int32` (no `time.Sleep`). Test helpers (`newXxx`, `assertXxx` with `t.Helper()`, `t.Cleanup`).

### references/concurrency.md

`sync.WaitGroup.Go()` (Go 1.25) replaces Add/goroutine/Done triplet. Context propagation rules (first parameter; never store except in lifecycle coordinators). Lifecycle Coordinator pattern with full excerpt from `~/code/agent-lab/pkg/lifecycle/lifecycle.go`. Graceful shutdown with timeout (`select` over `done` and `time.After`). Streaming response channels (return `<-chan T`, producer closes, respect context cancellation). Retry and backoff with context cancellation (excerpt from `~/tau/agent/client/retry.go:98-138`).

### references/modern-idioms.md

Version baseline: Go 1.26 (citing `go.work` line 1). `maps.Copy` (Go 1.21) — heavily used in tau/agent option merging. `slices` package (Go 1.21). `min`/`max` built-ins (Go 1.21) — citing retry backoff usage. `for range n` (Go 1.22). `sync.WaitGroup.Go` (Go 1.25). Range-over-function iterators (Go 1.23) — guidance to use only when a slice doesn't fit. `errors.Join` scope reminder pointing back to error-handling.md.

### references/registry-factory.md

When to use vs not. Registry structure (full `~/tau/provider/registry.go` excerpt — already verified, 60 lines). Factory function signatures: config-parameterized vs no-arg styles. Registration patterns: explicit `Register()` calls (tau convention) vs `init()` blank-import (classic Go). Thread safety via `sync.RWMutex`. End-to-end usage from `main.go`. Anti-patterns: missing mutex, returning internal map, factories that take non-config dependencies.

### references/ci-cd-release.md

Tag conventions table (`vX.Y.Z` / `module/vX.Y.Z` / `plugin/vX.Y.Z`). CHANGELOG.md format (`## [vX.Y.Z] - YYYY-MM-DD` with Breaking/Added/Changed/Fixed sections; parse-changelog compatible). Three full release.yml templates ready to copy: single-module (taiki-e + `v*`), multi-module (`v*` + `*/v*` with shell prefix extraction), marketplace plugin (`*/v*` only with `plugins/<name>/CHANGELOG.md`). Makefile release-tag targets.

### references/tooling-claude.md

Canonical Makefile with target matrix per project type (test/vet/build/run/dev/web). `.claude/settings.json` shape with `plansDirectory` and explicit Bash/Skill allow-list (full herald excerpt). CLAUDE.md conventions: mentorship mode vs lifecycle mode with selection criteria. `doc.go` conventions: one per package, contains only the package comment, structured `# Section` headers, indent-4 code examples (full `~/tau/agent/doc.go` excerpt as canonical example).

### references/workspace-multimodule.md

When to use a workspace. `go.work` structure (full `~/tau/go.work` already verified — 13 module paths in `use` block, including parent + child sub-modules listed independently). Submodule `go.mod` patterns (module path matches directory; sub-modules are how to escape the single-module depth budget for related-but-separate concerns). Sub-module decomposition criteria (separate go.mod is justified when: independent release cadence, independent dependency surface, plugin-style architecture, or the unit is meaningfully consumable in isolation). Versioning coordination across modules (release order, CHANGELOG cross-references). Workspace vs replace directive (workspace persistent and committed; replace per-module and removed before release). Cross-repo integration (external repos require published versions; local experimentation via developer-local replace).

## Critical Files to Create (12 new) and Modify (3)

**Create** (rewrite or new):
- `plugins/go-patterns/skills/go-patterns/SKILL.md` (rewrite)
- `plugins/go-patterns/skills/go-patterns/references/project-layout.md`
- `plugins/go-patterns/skills/go-patterns/references/package-design.md`
- `plugins/go-patterns/skills/go-patterns/references/configuration.md`
- `plugins/go-patterns/skills/go-patterns/references/error-handling.md`
- `plugins/go-patterns/skills/go-patterns/references/constructors.md`
- `plugins/go-patterns/skills/go-patterns/references/testing.md`
- `plugins/go-patterns/skills/go-patterns/references/concurrency.md`
- `plugins/go-patterns/skills/go-patterns/references/modern-idioms.md`
- `plugins/go-patterns/skills/go-patterns/references/registry-factory.md`
- `plugins/go-patterns/skills/go-patterns/references/ci-cd-release.md`
- `plugins/go-patterns/skills/go-patterns/references/tooling-claude.md`
- `plugins/go-patterns/skills/go-patterns/references/workspace-multimodule.md`

**Modify**:
- `plugins/go-patterns/.claude-plugin/plugin.json` (version `0.1.0` → `0.2.0`; expand description triggers)
- `plugins/go-patterns/CHANGELOG.md` (add `## [v0.2.0]` entry above existing `## [v0.1.0]`)
- (After PR approval, in a follow-up release session) tag `go-patterns/v0.2.0`.

## Source Files to Harvest Excerpts From

These are read-only inputs verified to exist at the cited paths/line ranges:

**Verified during planning**:
- `~/tau/go.work` (17 lines, all modules listed) ✓
- `~/tau/provider/registry.go` (60 lines, full Factory + registry + Register/Create/ListProviders) ✓
- `~/tau/agent/agent.go:1-100` (Agent interface lines 26-67, agent struct 69-76, New 78-92) ✓
- `~/tau/agent/errors.go` (162 lines, sentinels 23-28, AgentError 30-91, ErrorOptions 93-150, helpers 152-162) ✓

**To verify during implementation** (read first, then quote):
- `~/tau/agent/client/retry.go` — HTTPStatusError 15-28, isRetryableError 40-78, doWithRetry 98-138, min usage 86 & 95
- `~/tau/agent/agent.go:117-260` — Chat 117, ChatStream 139-146, mergeOptions 242-251
- `~/tau/agent/doc.go` — full file as canonical doc.go example
- `~/tau/protocol/config/agent.go` — Default 22-31, Merge 33-71, LoadAgentConfig 73-91
- `~/tau/provider/base.go` — full BaseProvider (~30 lines)
- `~/tau/format/registry.go` — no-arg Factory variant (~60 lines)
- `~/tau/format/openai/openai.go:12-15` — explicit Register() call style
- `~/tau/agent/mock/helpers.go` — builder constructors
- `~/tau/agent/tests/agent_test.go` — black-box test fixture style
- `~/tau/agent/CHANGELOG.md:1-20` — CHANGELOG format
- `~/tau/orchestrate/.github/workflows/release.yml` — single-module release.yml
- `~/tau/container/.github/workflows/release.yml` — multi-module release.yml
- `~/tau/tau-marketplace/.github/workflows/release.yml` — plugin marketplace release.yml
- `~/code/herald/internal/config/config.go:16-182` — env constants 16-25, Env structs 27-58, Config 61-70, Load 89-122, finalize 139-165, loadEnv 175-182
- `~/code/herald/cmd/server/main.go:27-35` — signal handling
- `~/code/herald/.claude/settings.json` — settings.json shape
- `~/code/herald/CLAUDE.md` — lifecycle-mode CLAUDE.md
- `~/code/go-agents/CLAUDE.md` — mentorship-mode CLAUDE.md
- `~/code/agent-lab/Makefile` — canonical Makefile
- `~/code/agent-lab/internal/infrastructure/infrastructure.go` — composition root example
- `~/code/agent-lab/pkg/lifecycle/lifecycle.go` — full Coordinator (86 lines)

## Existing Utilities and Patterns to Preserve

The current `SKILL.md` already names the eight core principles correctly. The rewrite preserves all eight names verbatim (Configuration is ephemeral, Interface-based layer interconnection, etc.) and the existing code examples (`chunk.ExtractContent()` semantic-getter contrast, `Execute(request ExecuteRequest)` parameter-encapsulation contrast). Nothing in v0.1.0 is wrong; it's just incomplete. The expansion adds depth, not corrections.

## Implementation Order

Execute in three batches to allow verification at natural breakpoints:

**Batch 1 — Frame**: SKILL.md, project-layout.md, package-design.md. Establishes vocabulary the rest depends on.

**Batch 2 — Behavior**: configuration.md, error-handling.md, constructors.md, testing.md, concurrency.md. The principles in action.

**Batch 3 — Tooling & Style**: modern-idioms.md, registry-factory.md, ci-cd-release.md, tooling-claude.md, workspace-multimodule.md. Style and infrastructure patterns.

After all three batches: bump `plugin.json` to `0.2.0`, prepend `CHANGELOG.md` entry, commit, push branch `update/go-patterns-deep-rewrite`, open PR.

## Verification

Confirm the rewrite is healthy:

1. **Structural**:
   - `wc -l plugins/go-patterns/skills/go-patterns/SKILL.md` → ≤250.
   - `find plugins/go-patterns/skills/go-patterns/references -name '*.md' | wc -l` → 12.
   - Every reference file >150 lines starts with a Table of Contents.
2. **Citations**:
   - `grep -R "Source:" plugins/go-patterns/skills/go-patterns/references/` returns paths that all exist on disk.
   - Code excerpts verified by spot-reading the cited files.
3. **Cross-references**:
   - Every `See: references/<file>.md#<heading>` resolves to an existing heading (manual scan).
4. **Skill triggers**:
   - SKILL.md description includes triggers for: interface design, error handling, package structure, configuration, CI/CD, multi-module workspaces, registry pattern.
5. **Plugin metadata**:
   - `cat plugins/go-patterns/.claude-plugin/plugin.json | jq .version` → `"0.2.0"`.
   - `head -20 plugins/go-patterns/CHANGELOG.md` shows new `## [v0.2.0]` entry above `## [v0.1.0]`.
6. **End-to-end skill load** (manual):
   - In a fresh Claude Code session, prompt: "design a new Go service with config and a release workflow" — confirm Claude consults the skill and loads only the relevant reference files (project-layout, configuration, ci-cd-release).
7. **PR**:
   - Branch: `update/go-patterns-deep-rewrite`.
   - Title: "go-patterns: deep rewrite with reference files for layered topics".
   - Body summarizes batches and links to CHANGELOG entry.
