# Release Session

## Purpose

Manage version releases for TAU ecosystem repositories. Two release types exist:

| Type | Trigger | Version Pattern | Example |
|------|---------|-----------------|---------|
| **Dev Release** | PR merged during a phase | `v<target>-dev.<objective>.<issue>` | `v0.1.0-dev.3.7` |
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
Objective #3: Kernel Core Loop (issues #7, #8, #12)

Issue #7 merged → v0.1.0-dev.3.7
Issue #8 merged → v0.1.0-dev.3.8
Issue #12 merged → v0.1.0-dev.3.12
Phase complete → v0.1.0
```

**Numbering**: Dev tags encode the objective and issue numbers from the GitHub
project hierarchy: `v<target>-dev.<objective-number>.<issue-number>`. Each tag
is deterministic — derived from the work item, not a sequential counter.

**Retention**: Dev releases accumulate during a phase and are cleaned up when
the phase release is tagged.

---

## Dev Release

Triggered after a PR is merged during active phase development. The CHANGELOG entry for this dev release was created during task execution closeout (Phase 8c). This session only handles tagging and verification.

### Step 1: Derive Tag from Issue Metadata

The dev tag is deterministic — derived from the objective and issue numbers:

```bash
# Format: v<target>-dev.<objective-number>.<issue-number>
# Example: working on Objective #3, Issue #7 for phase v0.1.0
TAG="v0.1.0-dev.3.7"
```

### Step 2: Tag and Push

```bash
git tag $TAG
git push origin main
git push origin $TAG
```

### Step 3: Verify

```bash
# Confirm the tag resolves on the Go module proxy
GOPROXY=https://proxy.golang.org go list -m <module>@<new-dev-tag>
```

---

## Phase Release

Triggered when all objectives in a phase are complete.

### Collect Session Metadata

- **Version**: Required, from argument (e.g., `/dev-workflow release v0.1.0`)
- Must follow semantic versioning with `v` prefix

### Step 1: Validate

Run the dev-type validation checklist. All quality gates must pass before proceeding.

### Step 2: Update CHANGELOG

Consolidate all `## v<target>-dev.*` sections into a single versioned entry and delete them from the CHANGELOG:

```markdown
## v0.1.0 - 2026-02-15
- All changes consolidated from dev releases...
```

After consolidation, no `## v<target>-dev.*` sections should remain in the file.

Commit CHANGELOG update:

```bash
git add CHANGELOG.md
git commit -m "release: <version>"
```

### Step 3: Archive Phase

If this release completes a phase:

```bash
# Archive the current phase document
PHASE_NUM=$(printf "%02d" <phase-number>)
mv _project/phase.md _project/.archive/phase.${PHASE_NUM}.md
```

Update `_project/README.md` subsystem status if subsystems moved from skeleton to complete during this phase.

### Step 4: Tag and Push

```bash
git tag <version>
git push origin main
git push origin <version>
```

### Step 5: Clean Up Dev Tags and Releases

Delete all dev tags (local and remote) and any associated GitHub releases for this phase:

```bash
git tag -l 'v<target>-dev.*' | while read tag; do
  gh release delete "$tag" --repo <owner>/<repo> --yes 2>/dev/null
  git push origin --delete "$tag"
  git tag -d "$tag"
done
```

### Step 6: Verify

```bash
# Confirm the module resolves on the proxy
GOPROXY=https://proxy.golang.org go list -m <module>@<version>

# Confirm the tag exists on remote
git ls-remote --tags origin <version>
```

## Dev-Type Augmentations

Dev-type references may specify additional conventions:

| Convention | Example |
|-----------|---------|
| Tag format | Go modules require `v` prefix (`v1.0.0`) |
| Validation commands | `go vet ./...`, `go test ./...` |
| Post-release steps | Module proxy warmup, documentation updates |

## Outcomes

**Dev release:**
- Module tagged with dev pre-release derived from objective and issue numbers

**Phase release:**
- Module tagged with clean version
- Dev tag CHANGELOG sections consolidated into versioned entry
- All dev releases for the phase deleted
- All validation checks passed before tag was created
