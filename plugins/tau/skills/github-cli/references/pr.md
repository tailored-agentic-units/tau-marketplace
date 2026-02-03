# gh pr

## Create

```bash
# Basic PR
gh pr create --title "Fix timeout bug" --body "Resolves #42"

# PR with HEREDOC body
gh pr create --title "Add retry logic" --body "$(cat <<'EOF'
## Summary
- Added exponential backoff retry logic to HTTP client
- Configurable max retries and initial backoff

## Test Plan
- [ ] Unit tests pass
- [ ] Integration tests with Ollama
EOF
)"

# Draft PR
gh pr create --title "WIP: new feature" --draft

# Target specific base branch
gh pr create --base develop --title "Feature branch merge"

# With reviewers
gh pr create --title "Ready for review" --reviewer user1,user2

# With labels
gh pr create --title "Bug fix" --label bug --label urgent
```

## List

```bash
# Open PRs
gh pr list --state open

# PRs authored by me
gh pr list --author @me

# PRs requesting my review
gh pr list --search "review-requested:@me"

# JSON output
gh pr list --json number,title,headRefName,state,reviewDecision
```

## View

```bash
# View PR details
gh pr view 45

# View diff
gh pr diff 45

# View only changed file names
gh pr diff 45 --name-only

# View PR comments
gh api repos/{owner}/{repo}/pulls/45/comments

# JSON output
gh pr view 45 --json title,body,commits,files,reviews
```

## Review

```bash
# Approve
gh pr review 45 --approve

# Request changes
gh pr review 45 --request-changes --body "Please fix the error handling"

# Comment
gh pr review 45 --comment --body "Looks good, minor suggestions"
```

## Merge

```bash
# Squash merge (preferred for clean history)
gh pr merge 45 --squash

# Merge commit
gh pr merge 45 --merge

# Rebase
gh pr merge 45 --rebase

# Auto-merge when checks pass
gh pr merge 45 --auto --squash

# Delete branch after merge
gh pr merge 45 --squash --delete-branch
```

## Check Status

```bash
# View CI checks status
gh pr checks 45

# Wait for checks to complete
gh pr checks 45 --watch
```

## Status

```bash
# Summary of PRs relevant to you (created, review requested, current branch)
gh pr status

# Include merge conflict status
gh pr status --conflict-status
```

## Ready / Draft

```bash
# Mark draft PR as ready for review
gh pr ready 45

# Convert PR back to draft
gh pr ready 45 --undo
```

## Reopen

```bash
# Reopen a closed PR
gh pr reopen 45

# Reopen with comment
gh pr reopen 45 --comment "Reopening after fixing CI"
```
