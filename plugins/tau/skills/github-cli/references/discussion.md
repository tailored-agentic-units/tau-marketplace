# gh discussions (via GraphQL)

GitHub Discussions have no native `gh discussion` CLI subcommand.
All operations use `gh api graphql` with the Discussions GraphQL API.

> **Category management** (create, edit, delete) is **not available** via the API.
> Manage categories through the repository settings web UI.

## Query Categories

```bash
# List discussion categories for a repo
gh api graphql -F owner='<owner>' -F repo='<repo>' -f query='
  query($owner: String!, $repo: String!) {
    repository(owner: $owner, name: $repo) {
      discussionCategories(first: 25) {
        nodes { id name emoji description isAnswerable }
      }
    }
  }'

# Get a specific category ID by name
gh api graphql -F owner='<owner>' -F repo='<repo>' -f query='
  query($owner: String!, $repo: String!) {
    repository(owner: $owner, name: $repo) {
      discussionCategories(first: 25) {
        nodes { id name }
      }
    }
  }' --jq '.data.repository.discussionCategories.nodes[] | select(.name == "Architecture") | .id'
```

## Create Discussion

```bash
# Get required IDs first
REPO_ID=$(gh api /repos/<owner>/<repo> --jq '.node_id')
CATEGORY_ID=$(gh api graphql -F owner='<owner>' -F repo='<repo>' -f query='
  query($owner: String!, $repo: String!) {
    repository(owner: $owner, name: $repo) {
      discussionCategories(first: 25) {
        nodes { id name }
      }
    }
  }' --jq '.data.repository.discussionCategories.nodes[] | select(.name == "Architecture") | .id')

# Create the discussion
gh api graphql \
  -f repositoryId="$REPO_ID" \
  -f categoryId="$CATEGORY_ID" \
  -f title="Discussion title" \
  -f body="Discussion body in **markdown**" \
  -f query='
  mutation($repositoryId: ID!, $categoryId: ID!, $title: String!, $body: String!) {
    createDiscussion(input: {
      repositoryId: $repositoryId
      categoryId: $categoryId
      title: $title
      body: $body
    }) {
      discussion { id url number title }
    }
  }'
```

## List Discussions

```bash
# List recent discussions
gh api graphql -F owner='<owner>' -F repo='<repo>' -f query='
  query($owner: String!, $repo: String!) {
    repository(owner: $owner, name: $repo) {
      discussions(first: 20, orderBy: {field: CREATED_AT, direction: DESC}) {
        nodes {
          number
          title
          url
          category { name }
          author { login }
          createdAt
          comments { totalCount }
        }
      }
    }
  }'

# Filter by category
gh api graphql -F owner='<owner>' -F repo='<repo>' -F categoryId='<category-id>' -f query='
  query($owner: String!, $repo: String!, $categoryId: ID!) {
    repository(owner: $owner, name: $repo) {
      discussions(first: 20, categoryId: $categoryId, orderBy: {field: CREATED_AT, direction: DESC}) {
        nodes { number title url author { login } createdAt }
      }
    }
  }'
```

## View Discussion

```bash
# View by number
gh api graphql -F owner='<owner>' -F repo='<repo>' -F number='<number>' -f query='
  query($owner: String!, $repo: String!, $number: Int!) {
    repository(owner: $owner, name: $repo) {
      discussion(number: $number) {
        id
        title
        body
        url
        category { name }
        author { login }
        createdAt
        comments(first: 20) {
          nodes { body author { login } createdAt }
        }
      }
    }
  }'
```

## Update Discussion

```bash
gh api graphql \
  -f discussionId="<discussion-node-id>" \
  -f title="Updated title" \
  -f body="Updated body" \
  -f query='
  mutation($discussionId: ID!, $title: String!, $body: String!) {
    updateDiscussion(input: {
      discussionId: $discussionId
      title: $title
      body: $body
    }) {
      discussion { id url title }
    }
  }'
```

## Close / Reopen Discussion

```bash
# Close
gh api graphql \
  -f discussionId="<discussion-node-id>" \
  -f query='
  mutation($discussionId: ID!) {
    closeDiscussion(input: { discussionId: $discussionId, reason: RESOLVED }) {
      discussion { id url title }
    }
  }'

# Reopen
gh api graphql \
  -f discussionId="<discussion-node-id>" \
  -f query='
  mutation($discussionId: ID!) {
    reopenDiscussion(input: { discussionId: $discussionId }) {
      discussion { id url title }
    }
  }'
```

## Comment on Discussion

```bash
gh api graphql \
  -f discussionId="<discussion-node-id>" \
  -f body="Comment in **markdown**" \
  -f query='
  mutation($discussionId: ID!, $body: String!) {
    addDiscussionComment(input: {
      discussionId: $discussionId
      body: $body
    }) {
      comment { id url }
    }
  }'
```

## Mark Answer (Q&A category only)

```bash
# Mark a comment as the answer
gh api graphql \
  -f commentId="<comment-node-id>" \
  -f query='
  mutation($commentId: ID!) {
    markDiscussionCommentAsAnswer(input: { id: $commentId }) {
      discussion { id title }
    }
  }'

# Unmark answer
gh api graphql \
  -f commentId="<comment-node-id>" \
  -f query='
  mutation($commentId: ID!) {
    unmarkDiscussionCommentAsAnswer(input: { id: $commentId }) {
      discussion { id title }
    }
  }'
```

## Delete Discussion

```bash
gh api graphql \
  -f discussionId="<discussion-node-id>" \
  -f query='
  mutation($discussionId: ID!) {
    deleteDiscussion(input: { id: $discussionId }) {
      discussion { id title }
    }
  }'
```
