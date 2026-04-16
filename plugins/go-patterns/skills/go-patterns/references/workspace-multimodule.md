# Workspace and Multi-Module Repos

When to use a Go workspace, the shape of `go.work`, sub-module decomposition
criteria, version coordination across related modules, the difference
between workspace and `replace` directive, and how external repos consume
workspace modules.

## Contents

1. [When to use a workspace](#when-to-use-a-workspace)
2. [`go.work` structure](#gowork-structure)
3. [Sub-module decomposition](#sub-module-decomposition)
4. [Sub-module `go.mod` patterns](#sub-module-gomod-patterns)
5. [Versioning coordination](#versioning-coordination)
6. [Workspace vs `replace` directive](#workspace-vs-replace-directive)
7. [Cross-repo integration](#cross-repo-integration)

## When to use a workspace

Use a Go workspace (`go.work`) when:

- Multiple Go modules are **developed together** in a single repository.
- Modules **depend on each other** at not-yet-published states (you change
  module A and want module B to immediately see those changes).
- You want **local edits to be visible across modules** without juggling
  `go mod edit -replace` lines.

Skip the workspace for:

- **Single-module repos** — there's nothing to coordinate.
- **Modules that release fully independently** with no shared development
  cadence.
- **CI environments** — CI should build against published versions, not
  workspace-resolved local paths. The `go.work` file is committed but
  `GOFLAGS=-mod=readonly` and similar can be set in CI to enforce
  published-only builds.

## `go.work` structure

The tau workspace coordinates 13 module paths:

```
go 1.26.1

use (
	./agent
	./container
	./container/docker
	./examples
	./format
	./format/converse
	./format/openai
	./orchestrate
	./protocol
	./provider
	./provider/azure
	./provider/bedrock
	./provider/ollama
)
```

Source: `/home/jaime/tau/go.work`

Syntax:

- **`go 1.26.1`** — the Go version line. Workspaces require Go 1.18+.
- **`use (...)` block** with paths relative to `go.work`, one per line.
- **Optional `replace` directives** apply to the entire workspace. Prefer
  per-module replaces when only one module needs an override.

Rules:

- Each path in `use` must contain a `go.mod`.
- **Parent modules and their sub-modules are listed independently.** Notice
  `./provider`, `./provider/azure`, `./provider/bedrock`, `./provider/ollama`
  all appear separately. The workspace doesn't infer sub-modules from
  parents.
- **Commit `go.work`.** Other developers need it to build your local
  topology.
- **`go.work.sum` should also be committed.** Go tooling maintains it
  alongside `go.work`.

## Sub-module decomposition

A sub-module is a directory with its own `go.mod`. Justifiable reasons to
extract one:

- **Independent release cadence.** The sub-module versions on its own
  schedule and consumers pin to its tags.
- **Distinct dependency surface.** `provider/azure` pulls in the Azure SDK;
  consumers of `provider/bedrock` should not have to download it. Splitting
  isolates dependencies.
- **Plugin-style architecture.** Consumers select which implementations to
  depend on. The registry pattern (see
  [registry-factory.md](registry-factory.md)) typically goes hand-in-hand
  with sub-module decomposition.
- **Meaningfully consumable in isolation.** A real downstream project would
  import this sub-module without the parent.

**Not** justifiable reasons:

- "I want a deeper directory structure." Sub-packages within a single
  module are limited to one level of depth (see
  [project-layout.md#depth-nuance-sub-packages-vs-sub-modules](project-layout.md#depth-nuance-sub-packages-vs-sub-modules)),
  but extracting a sub-module to dodge that rule trades a layout problem
  for a coordination problem.
- "These files belong together logically." That's what sub-packages are
  for.
- "I want to test these in isolation." That's what `package _test` and
  `tests/` mirroring do (see [testing.md](testing.md)).

Sub-modules add real coordination cost: separate `go.mod` to maintain,
separate CHANGELOG to write, separate tag to push, and explicit version
references in dependent modules. Use them when the benefit outweighs that
cost.

## Sub-module `go.mod` patterns

Each sub-module declares its own module path matching its directory:

```
module github.com/tailored-agentic-units/provider/azure

go 1.26

require (
	github.com/tailored-agentic-units/protocol v0.1.0
	github.com/tailored-agentic-units/provider v0.1.0
	// ... third-party deps
)
```

Key points:

- **Module path matches the directory** relative to the repo root:
  `github.com/<org>/<repo>/<sub>/<path>`.
- **Dependencies on sibling modules** use the **module path from that
  sibling's `go.mod`**, not relative imports. Even though they live in the
  same workspace, the import statement is the same as it would be from an
  external repo.
- **Go version** matches or is older than the parent.

Inside the workspace, `go build` resolves these requires to **local source
paths** (the `use` block in `go.work`). Outside the workspace (e.g., from a
consumer repo), the same requires resolve to **published versions** from
the module proxy. This duality is the entire reason workspaces exist.

## Versioning coordination

The challenge: module B depends on module A; both live in the same
workspace; both release independently.

The flow:

1. **Make breaking change in A on `main`.** Inside the workspace, B
   continues to build locally against A's unreleased source.
2. **Tag A.** `git tag protocol/v0.2.0; git push origin protocol/v0.2.0`.
   The release workflow (see [ci-cd-release.md](ci-cd-release.md)) creates
   the GitHub Release.
3. **Update B to depend on the new A.** `cd agent && go get
   github.com/tailored-agentic-units/protocol@v0.2.0`. This updates B's
   `go.mod` `require` line.
4. **Tag B.** `git tag agent/v0.2.0; git push origin agent/v0.2.0`. Note
   this may itself be a breaking version bump for B, depending on how A's
   change rippled.

CHANGELOG entries should cross-reference:

```markdown
## [v0.2.0] - 2026-04-16

### Breaking

- Requires protocol v0.2.0+ due to renamed MessageType field
```

For coordinated transitional work, use pre-release tags on A
(`protocol/v0.2.0-rc.1`) and let B pin to the rc while integration
proceeds. Replace with the final tag once the integration is stable.

## Workspace vs `replace` directive

Both mechanisms can redirect a module reference to a local path. They
differ in scope and persistence:

**`go.work`**:

- **Persistent and committed** to the repo.
- Applies to **all modules** in the workspace.
- Does **not** modify any `go.mod` file.
- Requires Go 1.18+.

**`replace` in `go.mod`**:

- **Per-module.** Affects only the module that declares it.
- **Persists in `go.mod`** — must be removed before release because the
  module proxy can't resolve local paths.
- Useful for **one-off overrides** during a bug fix in progress.

When to reach for `replace`:

- You're integrating with an **external local repo** that's not part of the
  workspace.
- You need to **temporarily override** a published version with a local
  fork while you submit and wait for an upstream patch.

When **not** to use `replace`:

- As a workspace substitute. It doesn't scale to many modules, and you'll
  forget to remove a `replace` line at release time.
- For long-lived local paths. Local paths break CI builds and can't be
  pushed to the module proxy.

## Cross-repo integration

External repos consume workspace modules by importing published versions:

- The external repo does **not** join the workspace (they're different
  repositories).
- The external repo's `go.mod` has `require
  github.com/tailored-agentic-units/protocol v0.2.0` pointing at a published
  tag.
- Example: `/home/jaime/code/herald/go.mod` requires
  `github.com/JaimeStill/go-agents` as a published dependency, not via
  workspace inclusion.

For local experimentation across repo boundaries:

- Add a developer-local `replace` to the consuming repo's `go.mod`:
  `replace github.com/JaimeStill/go-agents => ../go-agents`.
- Commit code without the `replace`; keep the `replace` only in untracked
  local work.
- Alternative: temporarily extend the producing repo's `go.work` to include
  the consuming repo. Not recommended for long-lived branches because it
  silently changes how that repo builds.

The cleanest workflow for cross-repo work in this ecosystem: tag a
pre-release of the producing module, pin the consuming repo to the
pre-release, then promote both to non-pre-release tags once the integration
is validated.
