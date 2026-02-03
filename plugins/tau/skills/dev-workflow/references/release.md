# Release Session

## Purpose

Finalize a version release by converting accumulated CHANGELOG entries, running validation, and creating a git tag.

## Prerequisites

- Load skills: **tau:github-cli**, plus domain-specific skills based on development type
- Collect session metadata: version (required, from argument)
- Ensure working tree is clean (`git status` shows no uncommitted changes)
- Ensure the current branch is `main`

## CHANGELOG Convention

Task execution sessions accumulate changes under a `## Current` header in `CHANGELOG.md`. The release workflow converts this header into a versioned entry.

**CHANGELOG format:**

```markdown
# Changelog

## Current

- Added audio protocol support (#42)
- Fixed embeddings response parsing (#45)

## v0.1.0 - 2026-01-15

- Initial release with Chat, Vision, Tools, and Embeddings protocols
```

If `## Current` does not exist or is empty, analyze the git history since the last tag to determine the CHANGELOG entries:

```bash
# Find the last tag
git describe --tags --abbrev=0

# List commits since last tag
git log $(git describe --tags --abbrev=0)..HEAD --oneline
```

## Workflow

### Step 1: Validate

Run the dev-type validation checklist to confirm the codebase is in a releasable state. All quality gates must pass before proceeding.

### Step 2: Update CHANGELOG

Replace the `## Current` header with the version and current date:

```markdown
## v0.2.0 - 2026-02-03
```

If `## Current` does not exist, create the version entry from git history and insert it after the `# Changelog` heading.

Commit the CHANGELOG update:

```bash
git add CHANGELOG.md
git commit -m "release: v0.2.0"
```

### Step 3: Tag and Push

```bash
git tag <version>
git push origin main
git push origin <version>
```

Push the branch first, then the tag, so the remote has the CHANGELOG commit before the tag points to it.

### Step 4: Verify

```bash
# Confirm the tag exists on remote
git ls-remote --tags origin <version>
```

## Dev-Type Augmentations

Dev-type references may specify additional conventions for the release workflow:

| Convention | Example |
|-----------|---------|
| Tag format | Go modules require `v` prefix (`v1.0.0`) |
| Validation commands | `go vet ./...`, `go test ./tests/...` |
| Post-release steps | Module proxy warmup, documentation updates |

## Outcomes

At the end of a release session:

- `CHANGELOG.md` has a versioned entry replacing `## Current`
- A git tag exists locally and on the remote
- All validation checks passed before the tag was created
