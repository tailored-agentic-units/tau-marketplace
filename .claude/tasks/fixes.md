# Fixes

## `gh issue create` does not support `--json` output flag

`gh issue create` prints the issue URL to stdout directly. The `--json url --jq '.url'` suffix causes an `unknown flag: --json` error. Remove it and capture stdout instead.

**Files to fix:**

| File | Line | Current | Fix |
|------|------|---------|-----|
| `plugins/tau/skills/dev-workflow/commands/objective.md` | 166 | `)" --json url --jq '.url')` | `)")` |
| `plugins/tau/skills/dev-workflow/commands/phase.md` | 147 | `)" --json url --jq '.url')` | `)")` |
| `plugins/tau/skills/project-management/references/objective-management.md` | 179 | `)" --json url --jq '.url')` | `)")` |
| `plugins/tau/skills/project-management/references/objective-management.md` | 208 | `)" --json url --jq '.url')` | `)")` |

**Not affected:** `composite-workflows.md:43` uses `gh issue list` which does support `--json`.

## `addSubIssue` mutation uses `subIssueUrl` (URI!) — type mismatch with String

The GraphQL `addSubIssue` mutation declares `subIssueUrl` as `URI!`, but `gh api graphql -f` passes all values as `String`, causing a type mismatch error. Use `subIssueId` (ID!) instead — resolve the child issue's node ID first via `gh issue view <url> --json id --jq '.id'`, then pass it as `subIssueId`.

**Files to fix:**

| File | Line | Current | Fix |
|------|------|---------|-----|
| `plugins/tau/skills/github-cli/references/sub-issue.md` | 43 | `subIssueUrl: $childUrl` | `subIssueId: $childId` |
| `plugins/tau/skills/project-management/references/objective-management.md` | 59 | `subIssueUrl: $childUrl` | `subIssueId: $childId` |
| `plugins/tau/skills/project-management/references/objective-management.md` | 224 | `subIssueUrl: $childUrl` | `subIssueId: $childId` |
| `plugins/tau/skills/dev-workflow/commands/objective.md` | 182 | `subIssueUrl: $childUrl` | `subIssueId: $childId` |

Each fix also requires updating the variable declaration (`$childUrl: URI!` → `$childId: ID!`) and the `-f` argument (`-f childUrl=` → `-f childId=`). The `sub-issue.md` reference doc at line 136-137 already documents both options — just update the example to prefer `subIssueId`.
