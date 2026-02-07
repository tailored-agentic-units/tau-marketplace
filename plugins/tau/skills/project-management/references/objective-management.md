# Objective Management

Objectives are parent issues that aggregate sub-issues across one or more repositories.
They represent a cohesive set of requirements within a phase subsection.

## Concepts

- An Objective is created on the **primary repository** for the body of work
- Sub-issues are created on their **respective repositories** and linked via GraphQL API
- Each sub-issue maps to exactly **one branch and one PR**
- Development sessions focus on resolving **one sub-issue at a time**
- Objectives and their sub-issues are defined during **concept development**

## Prerequisite: GraphQL Features Header

Sub-issues are a preview feature. All GraphQL operations require the header:

```
-H "GraphQL-Features: sub_issues"
```

## Get Issue Node ID

All mutations require the GraphQL node ID (not the issue number).

```bash
# From issue number (current repo)
gh issue view <number> --json id --jq '.id'

# From issue number (specific repo)
gh issue view <number> --repo <owner>/<repo> --json id --jq '.id'
```

## Add Sub-Issue

Link an existing issue as a sub-issue of an objective.

```bash
PARENT_ID=$(gh issue view <parent-number> --repo <owner>/<parent-repo> --json id --jq '.id')
CHILD_ID=$(gh issue view <child-number> --repo <owner>/<child-repo> --json id --jq '.id')

gh api graphql \
  -H "GraphQL-Features: sub_issues" \
  -f query='mutation($parentId: ID!, $childId: ID!) {
    addSubIssue(input: {issueId: $parentId, subIssueId: $childId}) {
      subIssue { number title url }
    }
  }' -f parentId="$PARENT_ID" -f childId="$CHILD_ID"
```

### Alternative: Add by URL

```bash
PARENT_ID=$(gh issue view <parent-number> --repo <owner>/<parent-repo> --json id --jq '.id')

gh api graphql \
  -H "GraphQL-Features: sub_issues" \
  -f query='mutation($parentId: ID!, $childUrl: URI!) {
    addSubIssue(input: {issueId: $parentId, subIssueUrl: $childUrl}) {
      subIssue { number title url }
    }
  }' -f parentId="$PARENT_ID" -f childUrl="https://github.com/<owner>/<repo>/issues/<number>"
```

### Replace Parent

Move a sub-issue from one objective to another:

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

Query all sub-issues of an objective.

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

## Get Parent of an Issue

```bash
gh api graphql \
  -H "GraphQL-Features: sub_issues" \
  -f query='query($owner: String!, $repo: String!, $number: Int!) {
    repository(owner: $owner, name: $repo) {
      issue(number: $number) {
        parent {
          number
          title
          url
          repository { nameWithOwner }
        }
      }
    }
  }' -f owner="<owner>" -f repo="<repo>" -F number=<number>
```

## Remove Sub-Issue

Unlink a sub-issue from its objective (does not delete the issue).

```bash
PARENT_ID=$(gh issue view <parent-number> --repo <owner>/<parent-repo> --json id --jq '.id')
CHILD_ID=$(gh issue view <child-number> --repo <owner>/<child-repo> --json id --jq '.id')

gh api graphql \
  -H "GraphQL-Features: sub_issues" \
  -f query='mutation($parentId: ID!, $childId: ID!) {
    removeSubIssue(input: {issueId: $parentId, subIssueId: $childId}) {
      subIssue { number title }
    }
  }' -f parentId="$PARENT_ID" -f childId="$CHILD_ID"
```

## Reprioritize Sub-Issue

Reorder a sub-issue within the parent's list. The `afterId` places the sub-issue
after the specified sibling (omit to move to the top).

```bash
gh api graphql \
  -H "GraphQL-Features: sub_issues" \
  -f query='mutation($parentId: ID!, $childId: ID!, $afterId: ID) {
    reprioritizeSubIssue(input: {issueId: $parentId, subIssueId: $childId, afterId: $afterId}) {
      subIssue { number title }
    }
  }' -f parentId="$PARENT_ID" -f childId="$CHILD_ID" -f afterId="$AFTER_ID"
```

## Composite: Create Objective with Sub-Issues

End-to-end workflow for creating an objective and linking cross-repo sub-issues.

```bash
OWNER="tailored-agentic-units"

# 1. Create the objective (parent issue) on the primary repo
#    Objectives only receive the 'objective' label — no category or package labels
OBJECTIVE_URL=$(gh issue create \
  --repo "$OWNER/<primary-repo>" \
  --title "Objective: <title>" \
  --label objective \
  --milestone "<phase-name>" \
  --body "$(cat <<'EOF'
## Objective

[What this objective achieves]

## Sub-Issues

Created across repositories as needed.
EOF
)" --json url --jq '.url')

PARENT_ID=$(gh issue view "$OBJECTIVE_URL" --json id --jq '.id')

# 2. Assign the Objective issue type (requires admin:org scope)
#    Get the Objective type ID: gh api graphql -f query='{ organization(login: "'"$OWNER"'") { issueTypes(first: 25) { nodes { id name } } } }'
gh api graphql -f query='
mutation($issueId: ID!, $typeId: ID!) {
  updateIssueIssueType(input: { issueId: $issueId, issueTypeId: $typeId }) {
    issue { number issueType { name } }
  }
}' -f issueId="$PARENT_ID" -f typeId="<objective-type-id>"

# 3. Create sub-issues on their respective repos
#    Sub-issues receive category label(s) and package label(s) — not 'objective'
CHILD_URL=$(gh issue create \
  --repo "$OWNER/<child-repo>" \
  --title "<sub-issue title>" \
  --label "<category>" --label "<package>" \
  --milestone "<phase-name>" \
  --body "$(cat <<'EOF'
## Context

[Context and scope for this sub-issue]

## Acceptance Criteria

- [ ] [Criteria]
EOF
)" --json url --jq '.url')

CHILD_ID=$(gh issue view "$CHILD_URL" --json id --jq '.id')

# 4. Assign the Task issue type to the sub-issue
gh api graphql -f query='
mutation($issueId: ID!, $typeId: ID!) {
  updateIssueIssueType(input: { issueId: $issueId, issueTypeId: $typeId }) {
    issue { number issueType { name } }
  }
}' -f issueId="$CHILD_ID" -f typeId="<task-type-id>"

# 5. Link sub-issue to objective
gh api graphql \
  -H "GraphQL-Features: sub_issues" \
  -f query='mutation($parentId: ID!, $childUrl: URI!) {
    addSubIssue(input: {issueId: $parentId, subIssueUrl: $childUrl}) {
      subIssue { number title url }
    }
  }' -f parentId="$PARENT_ID" -f childUrl="$CHILD_URL"

# 6. Add objective to project board and assign phase
# (see backlog-management.md and phase assignment patterns)
```
