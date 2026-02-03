# ID Resolution Patterns

The `gh project item-edit` command requires opaque IDs, not human-readable names. This reference documents how to resolve each ID type.

## ID Chain Overview

```
project number --> project ID       (gh project view --format json)
                   field name --> field ID     (gh project field-list --format json)
                                  option name --> option ID  (from field-list options)
item URL -------> item ID           (gh project item-add --format json)
```

All IDs are opaque strings with type-specific prefixes:

| ID Type | Prefix | Example |
|---------|--------|---------|
| Project | `PVT_` | `PVT_kwHOA...` |
| Field (SINGLE_SELECT) | `PVTSSF_` | `PVTSSF_lAHOA...` |
| Option | `PVTSO_` | `PVTSO_...` |
| Item | `PVTI_` | `PVTI_lAHOA...` |

## Project ID

```bash
gh project view <number> --owner <owner> --format json --jq '.id'
# Returns: "PVT_kwHOA..."
```

## Field ID

```bash
gh project field-list <number> --owner <owner> --format json \
  --jq '.fields[] | select(.name == "Phase") | .id'
# Returns: "PVTSSF_lAHOA..."
```

## Option ID

```bash
gh project field-list <number> --owner <owner> --format json \
  --jq '.fields[] | select(.name == "Phase") | .options[] | select(.name == "MVP Phase 1") | .id'
# Returns: "PVTSO_..."
```

## Item ID

```bash
# From item-add output
gh project item-add <number> --owner <owner> --url <issue-url> --format json --jq '.id'
# Returns: "PVTI_lAHOA..."

# From item-list by title
gh project item-list <number> --owner <owner> --format json \
  --jq '.items[] | select(.content.title == "Bug: login timeout") | .id'

# From item-list by issue number and repo
gh project item-list <number> --owner <owner> --format json \
  --jq '.items[] | select(.content.number == 42 and .content.repository == "owner/repo") | .id'
```

## Optimized ID Resolution

Resolve all IDs needed for a phase assignment with minimal API calls:

```bash
# 2 API calls instead of 3: project view + field-list (reuse field-list output)
PROJECT_ID=$(gh project view <number> --owner <owner> --format json --jq '.id')
FIELDS_JSON=$(gh project field-list <number> --owner <owner> --format json)
FIELD_ID=$(echo "$FIELDS_JSON" | jq -r '.fields[] | select(.name == "Phase") | .id')
OPTION_ID=$(echo "$FIELDS_JSON" | jq -r '.fields[] | select(.name == "Phase") | .options[] | select(.name == "TARGET_PHASE") | .id')
```

## Complete Phase Assignment Example

Putting it all together -- resolve all IDs and assign a phase in one script:

```bash
# Resolve project ID
PROJECT_ID=$(gh project view <number> --owner <owner> --format json --jq '.id')

# Resolve field and option IDs from a single field-list call
FIELDS_JSON=$(gh project field-list <number> --owner <owner> --format json)
FIELD_ID=$(echo "$FIELDS_JSON" | jq -r '.fields[] | select(.name == "Phase") | .id')
OPTION_ID=$(echo "$FIELDS_JSON" | jq -r '.fields[] | select(.name == "Phase") | .options[] | select(.name == "MVP Phase 1") | .id')

# Resolve item ID
ITEM_ID=$(gh project item-list <number> --owner <owner> --format json \
  --jq '.items[] | select(.content.number == 42) | .id')

# Assign the phase
gh project item-edit \
  --id "$ITEM_ID" \
  --project-id "$PROJECT_ID" \
  --field-id "$FIELD_ID" \
  --single-select-option-id "$OPTION_ID"
```
