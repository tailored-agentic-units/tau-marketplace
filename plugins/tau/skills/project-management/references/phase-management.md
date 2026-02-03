# Phase Management

Phases are SINGLE_SELECT fields on the project, typically named "Phase".

## Create the Phase Field

```bash
# Create with initial options
gh project field-create <number> --owner <owner> \
  --name "Phase" \
  --data-type SINGLE_SELECT \
  --single-select-options "Backlog,MVP Phase 1,v0.0.2,Done"
```

## List Fields and Phase Options

```bash
# List all fields
gh project field-list <number> --owner <owner> --format json

# Get the Phase field ID
gh project field-list <number> --owner <owner> --format json \
  --jq '.fields[] | select(.name == "Phase") | .id'

# Get all Phase option IDs
gh project field-list <number> --owner <owner> --format json \
  --jq '.fields[] | select(.name == "Phase") | .options'

# Get a specific option ID by name
gh project field-list <number> --owner <owner> --format json \
  --jq '.fields[] | select(.name == "Phase") | .options[] | select(.name == "MVP Phase 1") | .id'
```

## Add Phase Options

The CLI does not support adding individual options to an existing SINGLE_SELECT field.
To add options, delete and recreate the field.

```bash
# 1. Get the field ID
FIELD_ID=$(gh project field-list <number> --owner <owner> --format json \
  --jq '.fields[] | select(.name == "Phase") | .id')

# 2. Delete the old field
gh project field-delete --id "$FIELD_ID"

# 3. Recreate with all options (existing + new)
gh project field-create <number> --owner <owner> \
  --name "Phase" \
  --data-type SINGLE_SELECT \
  --single-select-options "Backlog,MVP Phase 1,v0.0.2,Sprint 3,Done"
```

> **Warning**: Deleting the field clears all existing phase assignments on items.
> Use this only when setting up for the first time or when a fresh start is acceptable.
> For non-destructive option management, use the GitHub web UI or the GraphQL API.

## Delete a Phase Field

```bash
FIELD_ID=$(gh project field-list <number> --owner <owner> --format json \
  --jq '.fields[] | select(.name == "Phase") | .id')
gh project field-delete --id "$FIELD_ID"
```
