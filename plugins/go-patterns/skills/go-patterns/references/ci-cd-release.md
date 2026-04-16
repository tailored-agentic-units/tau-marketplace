# CI/CD and Release

Tag conventions for single-module, multi-module, and marketplace-plugin
repos; the CHANGELOG.md format that `parse-changelog` (used by
`taiki-e/create-gh-release-action@v1`) understands; three full release
workflow templates ready to copy; and Makefile targets for tag creation.

## Contents

1. [Tag conventions](#tag-conventions)
2. [CHANGELOG.md format](#changelogmd-format)
3. [Single-module release workflow](#single-module-release-workflow)
4. [Multi-module release workflow](#multi-module-release-workflow)
5. [Marketplace plugin release workflow](#marketplace-plugin-release-workflow)
6. [Makefile integration](#makefile-integration)

## Tag conventions

Three tag formats are in use across this ecosystem:

| Format | Where used | Example |
|---|---|---|
| `vX.Y.Z` | Single-module repos | `v0.1.1` |
| `module/vX.Y.Z` | Multi-module repos (workspace root + sub-modules) | `azure/v0.2.0` |
| `plugin/vX.Y.Z` | Marketplace-style monorepos with `plugins/<name>/` | `go-patterns/v0.3.0` |

Rules:

- **Semantic versioning**: `MAJOR.MINOR.PATCH`.
- **`v` prefix required.** Go module proxy convention.
- **Pre-release format**: `v0.2.0-rc.1`. The release workflow excludes
  pre-release tags from the `latest` Docker tag (see Herald's
  `metadata-action` filter).
- **Tag prefix matches the relative directory** of the module in the repo.
  `azure/v0.2.0` releases the module rooted at `azure/`.

## CHANGELOG.md format

The `parse-changelog` tool (used by `taiki-e/create-gh-release-action@v1`)
extracts release notes from a CHANGELOG file. The accepted format:

```markdown
# Changelog

## [v0.1.1] - 2026-04-06

### Fixed

- Fix nil pointer dereference in streaming responses when format returns
  `(nil, nil)` for unrecognized stream events

## [v0.1.0] - 2026-04-06

### Added

- Agent interface with Chat, ChatStream, Vision, VisionStream, Tools, and Embed methods
- Explicit dependency injection constructor: `New(cfg, provider, format)`
- Client package with HTTP orchestration, retry logic, and health tracking
- Request package with ChatRequest, VisionRequest, ToolsRequest, and EmbeddingsRequest

### Changed

- ...

### Fixed

- ...

### Breaking

- ...
```

Source: `/home/jaime/tau/agent/CHANGELOG.md:1-20`

Rules:

- **Heading format**: `## [vX.Y.Z] - YYYY-MM-DD`. The bracketed semver and
  ISO date are required for `parse-changelog`'s anchoring.
- **Section order**: `Breaking`, `Added`, `Changed`, `Fixed`. (Some projects
  use `Deprecated` and `Removed` as well — include only sections that have
  entries.)
- **Backticks for identifiers**, code references, and configuration keys.
- **Latest at top, oldest at bottom.**
- **No "Unreleased" section between releases.** Add entries at tag time so
  every commit between tags is fully releasable.

The marketplace plugin convention adds the plugin prefix for
disambiguation: when the multi-module workflow tags
`go-patterns/v0.2.0`, `parse-changelog` looks at
`plugins/go-patterns/CHANGELOG.md` and the `## [v0.2.0]` heading inside.

## Single-module release workflow

For repos with a single Go module (most libraries, services, and CLIs):

```yaml
name: Release

on:
  push:
    tags:
      - 'v*'

permissions:
  contents: write

jobs:
  release:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Create GitHub Release
        uses: taiki-e/create-gh-release-action@v1
        with:
          token: ${{ secrets.GITHUB_TOKEN }}
          changelog: CHANGELOG.md
```

Source: `/home/jaime/tau/orchestrate/.github/workflows/release.yml`

Annotations:

- **`fetch-depth: 0`** required so the action can compare against previous
  tags.
- **`permissions: contents: write`** required to create the release.
- **The action extracts the section matching the pushed tag** from
  CHANGELOG.md automatically. No prefix needed.

Projects using this exact form: `tau/protocol`, `tau/orchestrate`,
`tau/agent`, `go-agents`, `go-agents-orchestration`.

## Multi-module release workflow

For repos with one Go module at the root **and** sub-modules with their own
`go.mod` (e.g., `tau/format` plus `tau/format/openai`,
`tau/format/converse`):

```yaml
name: Release

on:
  push:
    tags:
      - 'v*'
      - '*/v*'

permissions:
  contents: write

jobs:
  release:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Determine module from tag
        id: module
        run: |
          TAG="${GITHUB_REF#refs/tags/}"
          if [[ "$TAG" == */v* ]]; then
            PREFIX="${TAG%/v*}"
            echo "prefix=$PREFIX/" >> "$GITHUB_OUTPUT"
            echo "changelog=$PREFIX/CHANGELOG.md" >> "$GITHUB_OUTPUT"
          else
            echo "prefix=" >> "$GITHUB_OUTPUT"
            echo "changelog=CHANGELOG.md" >> "$GITHUB_OUTPUT"
          fi

      - name: Create GitHub Release
        uses: taiki-e/create-gh-release-action@v1
        with:
          token: ${{ secrets.GITHUB_TOKEN }}
          changelog: ${{ steps.module.outputs.changelog }}
          prefix: ${{ steps.module.outputs.prefix }}
```

Source: `/home/jaime/tau/container/.github/workflows/release.yml`

Annotations:

- **Two tag patterns trigger the workflow**: bare `v*` (root module),
  `*/v*` (sub-module).
- **Shell logic extracts the prefix** from the tag. For
  `azure/v0.2.0`, `PREFIX=azure`, `changelog=azure/CHANGELOG.md`.
- **The `prefix` parameter** tells `taiki-e/create-gh-release-action` to
  treat `azure/v0.2.0` as version `v0.2.0` of the `azure` module — the
  release name and asset names are scoped accordingly.
- **Each sub-module has its own `CHANGELOG.md`** at the sub-module root.

Projects using this pattern: `tau/format`, `tau/provider`, `tau/container`.

## Marketplace plugin release workflow

For Claude Code marketplace repos where each plugin lives at
`plugins/<name>/`:

```yaml
name: Release

on:
  push:
    tags:
      - '*/v*'

permissions:
  contents: write

jobs:
  release:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Determine plugin from tag
        id: plugin
        run: |
          TAG="${GITHUB_REF_NAME}"
          PREFIX="${TAG%/v*}"
          echo "prefix=$PREFIX/" >> "$GITHUB_OUTPUT"
          echo "changelog=plugins/$PREFIX/CHANGELOG.md" >> "$GITHUB_OUTPUT"

      - name: Create GitHub Release
        uses: taiki-e/create-gh-release-action@v1
        with:
          token: ${{ secrets.GITHUB_TOKEN }}
          prefix: ${{ steps.plugin.outputs.prefix }}
          changelog: ${{ steps.plugin.outputs.changelog }}
```

Source: `/home/jaime/tau/tau-marketplace/.github/workflows/release.yml`

Differences from the multi-module workflow:

- **Only `*/v*` tags trigger.** The marketplace itself doesn't have a
  root-level version.
- **CHANGELOG path includes `plugins/<name>/`.** Each plugin's CHANGELOG
  lives one directory deeper.

## Makefile integration

Add tag-creation targets to the canonical Makefile (see
[tooling-claude.md#makefile](tooling-claude.md#makefile) for the full
Makefile shape):

```makefile
.PHONY: tag tag-module

# Single-module: make tag VERSION=v0.2.0
tag:
	git tag -a $(VERSION) -m "Release $(VERSION)"
	git push origin $(VERSION)

# Multi-module: make tag-module MODULE=azure VERSION=v0.2.0
tag-module:
	git tag -a $(MODULE)/$(VERSION) -m "Release $(MODULE) $(VERSION)"
	git push origin $(MODULE)/$(VERSION)
```

Rules:

- **Tag and push in one make invocation.** Reduces "tagged but forgot to
  push" mistakes.
- **Annotated tags (`-a`) with a message** are required for some tooling.
- **Verify a clean working tree before tagging** — either manually or with
  a precondition target.
