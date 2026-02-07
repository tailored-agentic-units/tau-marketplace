# Issue Types (GraphQL API)

Issue types are an organization-level feature that categorize issues. The `gh` CLI does
not natively support issue types — all operations use the GraphQL API via `gh api graphql`.

> **Required scope**: Issue type mutations require the `admin:org` token scope.
> Add it with `gh auth refresh -s admin:org` or `gh auth login --scopes admin:org`.

## Standard Issue Types

| Type | Color | Purpose |
|------|-------|---------|
| **Bug** | RED | Something isn't working correctly |
| **Task** | YELLOW | Individual development task |
| **Objective** | BLUE | Parent issue that groups related work |

## Available Colors

`GRAY`, `BLUE`, `GREEN`, `YELLOW`, `ORANGE`, `RED`, `PINK`, `PURPLE`

## List Organization Issue Types

```bash
gh api graphql -f query='
{
  organization(login: "<owner>") {
    issueTypes(first: 25) {
      nodes {
        id
        name
        description
        color
        isEnabled
      }
    }
  }
}'
```

## Read Issue Type on an Issue

```bash
gh api graphql -f query='
query($owner: String!, $repo: String!, $number: Int!) {
  repository(owner: $owner, name: $repo) {
    issue(number: $number) {
      number
      title
      issueType { id name color }
    }
  }
}' -f owner="<owner>" -f repo="<repo>" -F number=<number>
```

### Bulk Read Types for All Open Issues

```bash
gh api graphql -f query='
{
  repository(owner: "<owner>", name: "<repo>") {
    issues(first: 50, states: OPEN) {
      nodes {
        number
        title
        issueType { name }
      }
    }
  }
}'
```

## Assign Issue Type

The `gh issue create` CLI does not support a `--type` flag. Assign types via GraphQL
after creating the issue.

```bash
ISSUE_ID=$(gh issue view <number> --repo <owner>/<repo> --json id --jq '.id')

gh api graphql -f query='
mutation($issueId: ID!, $typeId: ID!) {
  updateIssueIssueType(input: {
    issueId: $issueId
    issueTypeId: $typeId
  }) {
    issue { number title issueType { name } }
  }
}' -f issueId="$ISSUE_ID" -f typeId="<issue-type-id>"
```

### Remove Issue Type

Pass `null` for the type ID to clear the assignment:

```bash
gh api graphql -f query='
mutation($issueId: ID!) {
  updateIssueIssueType(input: {
    issueId: $issueId
    issueTypeId: null
  }) {
    issue { number title issueType { name } }
  }
}' -f issueId="$ISSUE_ID"
```

## Create Issue Type

Create a new type at the organization level.

```bash
# Get the organization node ID
ORG_ID=$(gh api graphql -f query='{ organization(login: "<owner>") { id } }' --jq '.data.organization.id')

gh api graphql -f query='
mutation($ownerId: ID!, $name: String!, $color: IssueTypeColor!) {
  createIssueType(input: {
    ownerId: $ownerId
    name: $name
    isEnabled: true
    description: "Description of the type"
    color: $color
  }) {
    issueType { id name color }
  }
}' -f ownerId="$ORG_ID" -f name="<type-name>" -f color="<COLOR>"
```

### CreateIssueTypeInput Fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `ownerId` | ID | Yes | Organization node ID |
| `name` | String | Yes | Display name |
| `isEnabled` | Boolean | Yes | Whether the type is active |
| `description` | String | No | Optional description |
| `color` | IssueTypeColor | No | Color enum value |

## Update Issue Type

Rename, recolor, or disable an existing type.

```bash
gh api graphql -f query='
mutation($typeId: ID!, $name: String, $color: IssueTypeColor) {
  updateIssueType(input: {
    issueTypeId: $typeId
    name: $name
    color: $color
  }) {
    issueType { id name description color isEnabled }
  }
}' -f typeId="<issue-type-id>" -f name="<new-name>" -f color="<COLOR>"
```

### Disable (Hide from UI)

```bash
gh api graphql -f query='
mutation($typeId: ID!) {
  updateIssueType(input: {
    issueTypeId: $typeId
    isEnabled: false
  }) {
    issueType { id name isEnabled }
  }
}' -f typeId="<issue-type-id>"
```

## Delete Issue Type

Permanently remove a type from the organization.

```bash
gh api graphql -f query='
mutation($typeId: ID!) {
  deleteIssueType(input: {
    issueTypeId: $typeId
  }) {
    clientMutationId
  }
}' -f typeId="<issue-type-id>"
```

## Limits

- Maximum **25 issue types** per organization
- Only **organization owners** can create, modify, or delete issue types
- Disabled types are hidden from the UI but preserved on existing issues
