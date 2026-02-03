# gh label

## Create

```bash
# Create with color and description
gh label create "bug" --color E99695 --description "Something isn't working"

# Create or update if exists
gh label create "enhancement" --color A2EEEF --description "New feature" --force
```

## List

```bash
# List all labels
gh label list

# Search by name or description
gh label list --search "bug"

# Sort by name
gh label list --sort name

# JSON output
gh label list --json name,color,description
```

## Edit

```bash
# Rename a label
gh label edit "bug" --name "defect"

# Update color and description
gh label edit "bug" --color FF0000 --description "Critical defect"
```

## Delete

```bash
gh label delete "stale" --yes
```

## Clone

```bash
# Clone labels from another repo into current repo
gh label clone owner/source-repo

# Overwrite existing labels
gh label clone owner/source-repo --force
```
