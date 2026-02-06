# Sub-Issues (GraphQL API)

Sub-issues create parent-child relationships between GitHub issues, including across
repositories within the same organization. The `gh` CLI does not natively support
sub-issues — all operations use the GraphQL API via `gh api graphql`.

> **Preview feature**: All sub-issue operations require the header
> `-H "GraphQL-Features: sub_issues"`.

## Get Issue Node ID

Mutations require the GraphQL node ID, not the issue number.

```bash
# Current repo
gh issue view <number> --json id --jq '.id'

# Specific repo
gh issue view <number> --repo <owner>/<repo> --json id --jq '.id'
```

## Add Sub-Issue

```bash
PARENT_ID=$(gh issue view <number> --repo <owner>/<repo> --json id --jq '.id')
CHILD_ID=$(gh issue view <number> --repo <owner>/<repo> --json id --jq '.id')

gh api graphql \
  -H "GraphQL-Features: sub_issues" \
  -f query='mutation($parentId: ID!, $childId: ID!) {
    addSubIssue(input: {issueId: $parentId, subIssueId: $childId}) {
      subIssue { number title url }
    }
  }' -f parentId="$PARENT_ID" -f childId="$CHILD_ID"
```

### Add by URL (alternative)

```bash
gh api graphql \
  -H "GraphQL-Features: sub_issues" \
  -f query='mutation($parentId: ID!, $childUrl: URI!) {
    addSubIssue(input: {issueId: $parentId, subIssueUrl: $childUrl}) {
      subIssue { number title url }
    }
  }' -f parentId="$PARENT_ID" -f childUrl="<issue-url>"
```

### Replace Parent

Move a sub-issue from one parent to another:

```bash
gh api graphql \
  -H "GraphQL-Features: sub_issues" \
  -f query='mutation($parentId: ID!, $childId: ID!) {
    addSubIssue(input: {issueId: $parentId, subIssueId: $childId, replaceParent: true}) {
      subIssue { number title url }
    }
  }' -f parentId="$NEW_PARENT_ID" -f childId="$CHILD_ID"
```

## List Sub-Issues

```bash
gh api graphql \
  -H "GraphQL-Features: sub_issues" \
  -f query='query($owner: String!, $repo: String!, $number: Int!) {
    repository(owner: $owner, name: $repo) {
      issue(number: $number) {
        title
        subIssues(first: 50) {
          nodes {
            number
            title
            state
            url
            repository { nameWithOwner }
          }
        }
        subIssuesSummary { total completed percentCompleted }
      }
    }
  }' -f owner="<owner>" -f repo="<repo>" -F number=<number>
```

## Get Parent

```bash
gh api graphql \
  -H "GraphQL-Features: sub_issues" \
  -f query='query($owner: String!, $repo: String!, $number: Int!) {
    repository(owner: $owner, name: $repo) {
      issue(number: $number) {
        parent { number title url repository { nameWithOwner } }
      }
    }
  }' -f owner="<owner>" -f repo="<repo>" -F number=<number>
```

## Remove Sub-Issue

Unlinks the sub-issue (does not delete it).

```bash
gh api graphql \
  -H "GraphQL-Features: sub_issues" \
  -f query='mutation($parentId: ID!, $childId: ID!) {
    removeSubIssue(input: {issueId: $parentId, subIssueId: $childId}) {
      subIssue { number title }
    }
  }' -f parentId="$PARENT_ID" -f childId="$CHILD_ID"
```

## Reprioritize Sub-Issue

Reorder within the parent's list. Omit `afterId` to move to the top.

```bash
gh api graphql \
  -H "GraphQL-Features: sub_issues" \
  -f query='mutation($parentId: ID!, $childId: ID!, $afterId: ID) {
    reprioritizeSubIssue(input: {issueId: $parentId, subIssueId: $childId, afterId: $afterId}) {
      subIssue { number title }
    }
  }' -f parentId="$PARENT_ID" -f childId="$CHILD_ID" -f afterId="$AFTER_ID"
```

## Input Types Reference

### AddSubIssueInput

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `issueId` | ID | Yes | Parent issue node ID |
| `subIssueId` | ID | No | Child issue node ID (provide this or `subIssueUrl`) |
| `subIssueUrl` | String | No | Child issue URL (alternative to `subIssueId`) |
| `replaceParent` | Boolean | No | Reassign child if it already has a parent |

### RemoveSubIssueInput

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `issueId` | ID | Yes | Parent issue node ID |
| `subIssueId` | ID | Yes | Child issue node ID |

### Issue Fields

| Field | Type | Description |
|-------|------|-------------|
| `parent` | Issue | The parent issue (null if none) |
| `subIssues` | IssueConnection | Paginated list of sub-issues |
| `subIssuesSummary` | SubIssuesSummary | `total`, `completed`, `percentCompleted` |
