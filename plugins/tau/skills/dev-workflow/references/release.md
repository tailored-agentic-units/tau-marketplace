# Release Session

## Purpose

Manage version releases across the TAU ecosystem. Two release types exist:

| Type | Trigger | Version Pattern | Example |
|------|---------|-----------------|---------|
| **Dev Release** | PR merged during a phase | `v<target>-dev.<NN>` | `v0.1.0-dev.03` |
| **Phase Release** | All phase objectives complete | `v<target>` | `v0.1.0` |

## Prerequisites

- Load skills: **tau:github-cli**, plus domain-specific skills based on development type
- Ensure working tree is clean (`git status` shows no uncommitted changes)
- Ensure the current branch is `main`

## Versioning Strategy

Phases correspond to semantic version targets. Dev pre-releases track incremental
progress as PRs are merged throughout the phase.

```
Phase Target: v0.1.0

PR #1 merged (tau-core)       → tau-core v0.1.0-dev.01
PR #2 merged (tau-agent)      → tau-agent v0.1.0-dev.01
PR #3 merged (tau-core)       → tau-core v0.1.0-dev.02
                                 tau-agent v0.1.0-dev.02  (cascade: go.mod update)
Phase complete                → tau-core v0.1.0, tau-agent v0.1.0
```

**Numbering**: Dev numbers are sequential per module per phase (dev.01, dev.02, ...).

**Retention**: Only current + previous dev releases are kept. Older dev releases
are deleted after each new dev tag.

---

## Dev Release

Triggered after a PR is merged during active phase development. Only tag modules
that changed in the PR, then cascade to dependent modules.

### Step 1: Identify Changed Modules

Determine which modules were modified in the merged PR:

```bash
# From main after merge
git diff HEAD~1 --name-only | cut -d/ -f1 | sort -u
```

### Step 2: Determine Next Dev Number

For each changed module, find the current dev number and increment:

```bash
cd <module-dir>
git tag -l 'v<target>-dev.*' --sort=-v:refname | head -1
# If v0.1.0-dev.02 → next is v0.1.0-dev.03
# If no tags exist → start at v0.1.0-dev.01
```

### Step 3: Tag in Dependency Order

Tag modules bottom-up through the dependency chain:

```
tau-core (no TAU deps)
  └→ tau-agent (depends on tau-core)
       └→ tau-orchestrate (depends on tau-agent)
```

For each module, in order:

```bash
cd <module-dir>

# Update go.mod if a dependency was just tagged
# (only needed for cascade — skip for the module that directly changed)
go get github.com/tailored-agentic-units/<dep>@<new-dev-tag>
go mod tidy

# Commit dependency update if go.mod changed
git add go.mod go.sum
git commit -m "deps: <dep> <new-dev-tag>"

# Tag and push
git tag <new-dev-tag>
git push origin main
git push origin <new-dev-tag>
```

### Step 4: Clean Up Old Dev Releases

After tagging, delete dev releases older than previous. For each tagged module:

```bash
# List dev tags for this phase target, oldest first
TAGS=$(git tag -l 'v<target>-dev.*' --sort=v:refname)

# Keep the last 2 (current + previous), delete the rest
echo "$TAGS" | head -n -2 | while read tag; do
  gh release delete "$tag" --repo <owner>/<repo> --yes 2>/dev/null
  git push origin --delete "$tag"
  git tag -d "$tag"
done
```

### Step 5: Verify

```bash
# Confirm the tag resolves on the Go module proxy
GOPROXY=https://proxy.golang.org go list -m <module>@<new-dev-tag>
```

---

## Phase Release

Triggered when all objectives in a phase are complete. Converts dev pre-releases
into clean versioned releases.

### Collect Session Metadata

- **Version**: Required, from argument (e.g., `/dev-workflow release v0.1.0`)
- Must follow semantic versioning with `v` prefix

### Step 1: Validate

Run the dev-type validation checklist across all modules that changed during
the phase. All quality gates must pass before proceeding.

### Step 2: Update CHANGELOGs

For each module being released, replace the `## Current` header with the version
and current date:

```markdown
## v0.1.0 - 2026-02-15
```

If `## Current` does not exist, analyze git history since the last release tag:

```bash
git describe --tags --abbrev=0
git log $(git describe --tags --abbrev=0)..HEAD --oneline
```

Commit CHANGELOG updates:

```bash
git add CHANGELOG.md
git commit -m "release: <version>"
```

### Step 3: Tag in Dependency Order

Same bottom-up dependency order as dev releases. For each module:

```bash
cd <module-dir>

# Update go.mod to use the clean version of dependencies
go get github.com/tailored-agentic-units/<dep>@<version>
go mod tidy

git add go.mod go.sum
git commit -m "deps: <dep> <version>"

git tag <version>
git push origin main
git push origin <version>
```

### Step 4: Clean Up Remaining Dev Releases

Delete all dev releases for this phase target:

```bash
git tag -l 'v<target>-dev.*' | while read tag; do
  gh release delete "$tag" --repo <owner>/<repo> --yes 2>/dev/null
  git push origin --delete "$tag"
  git tag -d "$tag"
done
```

### Step 5: Verify

```bash
# Confirm each module resolves on the proxy
GOPROXY=https://proxy.golang.org go list -m <module>@<version>

# Confirm the tag exists on remote
git ls-remote --tags origin <version>
```

## Dev-Type Augmentations

Dev-type references may specify additional conventions:

| Convention | Example |
|-----------|---------|
| Tag format | Go modules require `v` prefix (`v1.0.0`) |
| Validation commands | `go vet ./...`, `go test ./tests/...` |
| Post-release steps | Module proxy warmup, documentation updates |

## Outcomes

**Dev release:**
- Changed modules tagged with next dev pre-release
- Dependent modules cascaded and tagged
- Old dev releases cleaned up (current + previous retained)

**Phase release:**
- All phase modules tagged with clean version
- CHANGELOGs converted from `## Current` to versioned entries
- All dev releases for the phase deleted
- All validation checks passed before tags were created
