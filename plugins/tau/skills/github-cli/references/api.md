# gh api

## REST

```bash
# Get repository details
gh api repos/{owner}/{repo}

# List issue comments
gh api repos/{owner}/{repo}/issues/42/comments

# Create a comment (POST is implied when using -f or -F)
gh api repos/{owner}/{repo}/issues/42/comments -f body="Comment text"

# Paginated results
gh api repos/{owner}/{repo}/issues --paginate
```

## HTTP Methods

```bash
# Explicit GET request
gh api repos/{owner}/{repo}/pulls --method GET

# POST with JSON body
gh api repos/{owner}/{repo}/issues -f title="New issue" -f body="Issue description"

# PATCH to update a resource
gh api repos/{owner}/{repo}/issues/42 --method PATCH -f state="closed"

# DELETE a resource
gh api repos/{owner}/{repo}/git/refs/tags/v0.0.1 --method DELETE

# PUT to replace a resource (e.g., set topics)
gh api repos/{owner}/{repo}/topics --method PUT -f names[]="go" -f names[]="cli"
```

## Custom Headers

```bash
# Add custom Accept header (e.g., for preview APIs)
gh api repos/{owner}/{repo}/releases -H "Accept: application/vnd.github.v3+json"

# Request raw content
gh api repos/{owner}/{repo}/readme -H "Accept: application/vnd.github.raw+json"
```

## Request Body

```bash
# JSON fields with -f (string) and -F (typed: int, bool, null)
gh api repos/{owner}/{repo}/issues -f title="Bug report" -F draft=true -F milestone=3

# Raw JSON body from stdin
echo '{"title":"From JSON","body":"Created via raw JSON"}' | gh api repos/{owner}/{repo}/issues --input -

# Request body from file
gh api repos/{owner}/{repo}/issues --input request.json
```

## Error Response Handling

```bash
# Show HTTP status code alongside response
gh api repos/{owner}/{repo}/issues/99999 --include 2>/dev/null | head -1

# Silence errors and inspect exit code
gh api repos/{owner}/{repo}/issues/99999 --silent; echo "Exit code: $?"

# Extract specific error message with jq
gh api repos/{owner}/{repo}/issues/99999 2>&1 | jq -r '.message // .errors[0].message // "Unknown error"'
```

## Paginated Requests

```bash
# Auto-paginate all results
gh api repos/{owner}/{repo}/issues --paginate

# Paginate with jq to extract specific fields
gh api repos/{owner}/{repo}/issues --paginate --jq '.[].title'

# Limit pages with per_page parameter
gh api repos/{owner}/{repo}/issues?per_page=10&page=2
```

## GraphQL

```bash
gh api graphql -F owner='{owner}' -F name='{repo}' -f query='
  query($name: String!, $owner: String!) {
    repository(owner: $owner, name: $name) {
      releases(last: 3) {
        nodes { tagName }
      }
    }
  }
'
```
