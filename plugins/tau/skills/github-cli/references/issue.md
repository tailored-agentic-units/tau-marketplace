# gh issue

## Create

```bash
# Simple issue
gh issue create --title "Bug: login fails on timeout" --body "Steps to reproduce..."

# With labels and assignee
gh issue create --title "Add retry logic" --label enhancement --label "good first issue" --assignee @me

# Multi-line body using HEREDOC
gh issue create --title "Feature request" --body "$(cat <<'EOF'
## Summary
Description of the feature.

## Acceptance Criteria
- [ ] Criterion 1
- [ ] Criterion 2

## Context
Additional context here.
EOF
)"
```

## List

```bash
# Open issues assigned to me
gh issue list --state open --assignee @me

# Filter by label
gh issue list --label bug --state open

# JSON output with limit
gh issue list --json number,title,labels,state --limit 50

# Filter by milestone
gh issue list --milestone "v1.0"
```

## Search

```bash
# Search in current repo
gh search issues "timeout error" --repo $(gh repo view --json nameWithOwner -q .nameWithOwner)

# Search with filters
gh search issues "bug" --repo owner/repo --state open --label critical

# Search across an organization
gh search issues "memory leak" --owner my-org --state open
```

## View

```bash
# View issue details
gh issue view 42

# JSON output
gh issue view 42 --json title,body,labels,assignees,comments

# View in browser
gh issue view 42 --web
```

## Edit

```bash
# Add labels
gh issue edit 42 --add-label enhancement --add-label "help wanted"

# Remove label
gh issue edit 42 --remove-label bug

# Reassign
gh issue edit 42 --add-assignee username

# Update title and body
gh issue edit 42 --title "Updated title" --body "Updated description"

# Set milestone
gh issue edit 42 --milestone "v1.0"
```

## Close

```bash
# Close with comment
gh issue close 42 --comment "Fixed in #45"

# Close as not planned
gh issue close 42 --reason "not planned"
```
