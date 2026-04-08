# Composite Workflows

Multi-step operations that chain several `gh project` commands together.

## Bootstrap a New Project

Create a project, link repos, set up the Phase field, and bootstrap labels.

```bash
# 1. Create the project
PROJECT_NUM=$(gh project create --owner <owner> --title "TAU Platform" \
  --format json --jq '.number')

# 2. Link repositories
gh project link "$PROJECT_NUM" --owner <owner> --repo <owner>/tau-core
gh project link "$PROJECT_NUM" --owner <owner> --repo <owner>/tau-skills

# 3. Create the Phase field
gh project field-create "$PROJECT_NUM" --owner <owner> \
  --name "Phase" \
  --data-type SINGLE_SELECT \
  --single-select-options "Backlog,MVP Phase 1,v0.0.2,Done"

# 4. Set visibility
gh project edit "$PROJECT_NUM" --owner <owner> --visibility PRIVATE

# 5. Bootstrap standard labels on linked repos (see Label Convention)
gh label clone <owner>/tau-platform --repo <owner>/tau-core --force
gh label clone <owner>/tau-platform --repo <owner>/tau-skills --force

# 6. Create milestones for non-meta phases on each linked repo
for repo in <owner>/tau-core <owner>/tau-skills; do
  gh api --method POST "/repos/$repo/milestones" \
    -f title="MVP Phase 1" \
    -f description="Phase 1 -- see TAU Platform project board"
done
```

## Bulk Add Issues to Project

```bash
# Add all open issues from a repo
for url in $(gh issue list --repo <owner>/<repo> --state open --json url --jq '.[].url'); do
  gh project item-add <number> --owner <owner> --url "$url"
done
```

## View Phase Progress

```bash
# List items with their phase
gh project item-list <number> --owner <owner> -L 200 --format json \
  --jq '.items[] | {title: .content.title, repo: .content.repository, phase: (.fieldValues[] | select(.field.name == "Phase") | .name) // "Unassigned"}'

# Count items per phase
gh project item-list <number> --owner <owner> -L 200 --format json \
  --jq '[.items[] | (.fieldValues[] | select(.field.name == "Phase") | .name) // "Unassigned"] | group_by(.) | map({phase: .[0], count: length})'

# List items in a specific phase
gh project item-list <number> --owner <owner> -L 200 --format json \
  --jq '.items[] | select(any(.fieldValues[]; .field.name == "Phase" and .name == "MVP Phase 1")) | {title: .content.title, repo: .content.repository}'
```

## Move Items Between Phases

```bash
# Resolve IDs once
PROJECT_ID=$(gh project view <number> --owner <owner> --format json --jq '.id')
FIELD_ID=$(gh project field-list <number> --owner <owner> --format json \
  --jq '.fields[] | select(.name == "Phase") | .id')
NEW_OPTION_ID=$(gh project field-list <number> --owner <owner> --format json \
  --jq '.fields[] | select(.name == "Phase") | .options[] | select(.name == "v0.0.2") | .id')

# Move a single item
gh project item-edit \
  --id <item-id> \
  --project-id "$PROJECT_ID" \
  --field-id "$FIELD_ID" \
  --single-select-option-id "$NEW_OPTION_ID"

# Bulk move all items from one phase to another
OLD_PHASE="MVP Phase 1"
ITEM_IDS=$(gh project item-list <number> --owner <owner> -L 200 --format json \
  --jq ".items[] | select(any(.fieldValues[]; .field.name == \"Phase\" and .name == \"$OLD_PHASE\")) | .id")
for id in $ITEM_IDS; do
  gh project item-edit \
    --id "$id" \
    --project-id "$PROJECT_ID" \
    --field-id "$FIELD_ID" \
    --single-select-option-id "$NEW_OPTION_ID"
done
```

## Cross-Repo Backlog Overview

```bash
# Items grouped by repository
gh project item-list <number> --owner <owner> -L 200 --format json \
  --jq '[.items[] | {title: .content.title, repo: .content.repository, type: .content.type}] | group_by(.repo) | map({repo: .[0].repo, count: length, items: map(.title)})'
```
