# gh search

## Issues

```bash
gh search issues "timeout" --repo owner/repo --state open
gh search issues "bug" --label critical --state open --owner my-org
gh search issues --assignee @me --state open
```

## Pull Requests

```bash
gh search prs "fix" --review-requested=@me --state open
gh search prs --author @me --merged --repo owner/repo
gh search prs --checks failure --state open
```

## Code

```bash
gh search code "func NewAgent" --repo owner/repo
gh search code "TODO" --owner my-org --language go
gh search code "interface Provider" --extension go
```

## Date Filters

```bash
# Issues created after a specific date
gh search issues "bug" --repo owner/repo -- "created:>2024-01-01"

# Issues updated since a date
gh search issues "feature" --repo owner/repo -- "updated:>=2024-06-01"

# PRs merged within a date range
gh search prs --repo owner/repo --merged -- "merged:2024-01-01..2024-06-30"

# Issues created in the last 7 days
gh search issues "bug" --repo owner/repo -- "created:>=$(date -d '7 days ago' +%Y-%m-%d)"
```

## Sorting

```bash
# Sort issues by most recently created
gh search issues "bug" --repo owner/repo --sort created --order desc

# Sort issues by most recently updated
gh search issues "feature" --repo owner/repo --sort updated --order desc

# Sort repositories by stars
gh search repos "cli tool" --sort stars --order desc

# Sort code results by recently indexed
gh search code "TODO" --repo owner/repo --sort indexed --order desc
```

## Combined Filters

```bash
# Open bugs created this year, assigned to someone, sorted by update time
gh search issues "crash" --repo owner/repo --state open --label bug --sort updated -- "created:>2024-01-01"

# Merged PRs by a specific author in a date range
gh search prs --repo owner/repo --author username --merged --sort created -- "merged:2024-01-01..2024-12-31"

# Unreviewed open PRs requesting your review
gh search prs --review-requested=@me --state open --sort created --order asc
```

## JSON Output

```bash
# Output issue search results as JSON
gh search issues "bug" --repo owner/repo --json number,title,state,labels,createdAt

# Use --jq to filter JSON output
gh search issues "bug" --repo owner/repo --json number,title,labels --jq '.[].title'

# Extract specific fields with jq expressions
gh search prs --author @me --merged --json number,title,url --jq '.[] | "\(.number) \(.title)"'

# Get count of matching results
gh search issues "bug" --repo owner/repo --json number --jq 'length'
```
