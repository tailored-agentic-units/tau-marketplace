# GraphQL Heredoc Fix for dev-workflow Skill

## Context

GraphQL queries using inline `-f query='...'` syntax have `$` signs in variables mangled by Claude Code's Bash tool. The fix replaces all inline queries with heredoc syntax (`cat <<'GRAPHQL'`) which preserves `$` signs correctly.

## Changes (already applied)

All 6 occurrences across 2 files have been updated:

### `plugins/tau/skills/dev-workflow/commands/objective.md` (5 fixes)
1. **Lines ~54–66** — `query($id: ID!)` sub-issue status check
2. **Lines ~85–91** — `mutation($parentId, $childId)` removeSubIssue
3. **Lines ~170–175** — `mutation($issueId, $typeId)` updateIssueIssueType
4. **Lines ~179–185** — `mutation($parentId, $childId)` addSubIssue
5. **Lines ~192–198** — `mutation($parentId, $childId)` addSubIssue (replaceParent)

### `plugins/tau/skills/dev-workflow/commands/phase.md` (1 fix)
6. **Lines ~245–250** — `mutation($issueId, $typeId)` updateIssueIssueType

## Verification

Review the diffs to confirm each heredoc block is properly structured with matching `GRAPHQL` delimiters and that the `-f` variable flags remain on the closing line after the heredoc.
