# Fix gh CLI Issues in Plugin Skills

## Context

Two bugs were discovered during development sessions involving gh CLI calls in plugin skills:

1. **`gh issue create` doesn't support `--json`** — The command prints the issue URL to stdout directly. Appending `--json url --jq '.url'` causes `unknown flag: --json`. Fix: remove the flag and capture stdout.
2. **`addSubIssue` mutation `subIssueUrl` type mismatch** — `gh api graphql -f` passes all values as `String`, but the `subIssueUrl` parameter expects `URI!`, causing a type error. Fix: use `subIssueId` (type `ID!`) instead, which works with `-f` since GraphQL IDs are strings. The child issue's node ID is already resolved in the surrounding code via `gh issue view --json id --jq '.id'`.

## Changes

### Fix 1: Remove `--json url --jq '.url'` from `gh issue create`

| File | Line | Change |
|------|------|--------|
| `plugins/tau/skills/dev-workflow/commands/objective.md` | 166 | `)" --json url --jq '.url')` → `)")` |
| `plugins/tau/skills/dev-workflow/commands/phase.md` | 147 | `)" --json url --jq '.url')` → `)")` |
| `plugins/tau/skills/project-management/references/objective-management.md` | 179 | `)" --json url --jq '.url')` → `)")` |
| `plugins/tau/skills/project-management/references/objective-management.md` | 208 | `)" --json url --jq '.url')` → `)")` |

### Fix 2: Replace `subIssueUrl` with `subIssueId` in `addSubIssue` mutations

**a) Workflow files** — Change the mutation to use `$childId`/`subIssueId` (already resolved in surrounding code):

| File | Lines | Change |
|------|-------|--------|
| `plugins/tau/skills/dev-workflow/commands/objective.md` | 181-185 | `$childUrl: URI!` → `$childId: ID!`, `subIssueUrl: $childUrl` → `subIssueId: $childId`, `-f childUrl=` → `-f childId=` |
| `plugins/tau/skills/project-management/references/objective-management.md` | 223-227 | Same pattern |

**b) Reference docs** — Remove the broken "Add by URL (alternative)" sections. The primary "Add by ID" examples are already correct, and the Input Types Reference table still documents `subIssueUrl` as an available field:

| File | Lines | Change |
|------|-------|--------|
| `plugins/tau/skills/github-cli/references/sub-issue.md` | 37-47 | Remove "Add by URL (alternative)" section |
| `plugins/tau/skills/project-management/references/objective-management.md` | 51-63 | Remove "Alternative: Add by URL" section |

## Verification

After making changes, visually inspect each file to confirm:
- No `--json url --jq '.url'` remains on any `gh issue create` command
- All `addSubIssue` mutations use `subIssueId`/`$childId` (type `ID!`)
- No broken "Add by URL" alternative sections remain
- Surrounding code (ID resolution, issue type assignment) is intact
