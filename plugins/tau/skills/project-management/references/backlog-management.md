# Backlog Management

Add issues, create drafts, list items, assign phases, and archive operations.

## Add Issues to Project

```bash
# Add an issue by URL
gh project item-add <number> --owner <owner> \
  --url https://github.com/<owner>/<repo>/issues/42

# Add and capture the item ID
gh project item-add <number> --owner <owner> \
  --url https://github.com/<owner>/<repo>/issues/42 \
  --format json --jq '.id'
```

## Create Draft Items

```bash
# Create a draft issue directly on the project
gh project item-create <number> --owner <owner> \
  --title "Investigate performance regression" \
  --body "Details here..."
```

## List Items

```bash
# List all items (default 30)
gh project item-list <number> --owner <owner> --format json

# List up to 200 items
gh project item-list <number> --owner <owner> -L 200 --format json

# List with key details
gh project item-list <number> --owner <owner> --format json \
  --jq '.items[] | {id, title: .content.title, type: .content.type, repo: .content.repository}'
```

## Assign a Phase to an Item

Requires three resolved IDs: project ID, field ID, and option ID. See [id-resolution.md](id-resolution.md) for the full ID chain.

```bash
# Step 1: Get the project ID
PROJECT_ID=$(gh project view <number> --owner <owner> --format json --jq '.id')

# Step 2: Get the Phase field ID
FIELD_ID=$(gh project field-list <number> --owner <owner> --format json \
  --jq '.fields[] | select(.name == "Phase") | .id')

# Step 3: Get the option ID for the target phase
OPTION_ID=$(gh project field-list <number> --owner <owner> --format json \
  --jq '.fields[] | select(.name == "Phase") | .options[] | select(.name == "MVP Phase 1") | .id')

# Step 4: Get the item ID (if not already known)
ITEM_ID=$(gh project item-list <number> --owner <owner> --format json \
  --jq '.items[] | select(.content.title == "Issue title") | .id')

# Step 5: Set the phase
gh project item-edit \
  --id "$ITEM_ID" \
  --project-id "$PROJECT_ID" \
  --field-id "$FIELD_ID" \
  --single-select-option-id "$OPTION_ID"
```

## Clear a Phase Assignment

```bash
gh project item-edit \
  --id "$ITEM_ID" \
  --project-id "$PROJECT_ID" \
  --field-id "$FIELD_ID" \
  --clear
```

## Archive / Unarchive Items

```bash
# Archive a completed item
gh project item-archive <number> --owner <owner> --id <item-id>

# Unarchive
gh project item-archive <number> --owner <owner> --id <item-id> --undo
```

## Delete an Item

```bash
gh project item-delete <number> --owner <owner> --id <item-id>
```

> **Tip**: Prefer `item-archive` over `item-delete` to preserve history.
