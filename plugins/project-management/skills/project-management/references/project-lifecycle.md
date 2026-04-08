# Project Lifecycle

Create, list, view, edit, close, delete, and link repositories to GitHub Projects v2.

## Create

```bash
# Create a project
gh project create --owner <owner> --title "TAU Platform" --format json

# Create and capture the project number
gh project create --owner <owner> --title "TAU Platform" --format json --jq '.number'
```

## List

```bash
# List open projects
gh project list --owner <owner> --format json

# Include closed projects
gh project list --owner <owner> --closed --format json

# Extract numbers and titles
gh project list --owner <owner> --format json --jq '.projects[] | {number, title}'
```

## View

```bash
# View project details
gh project view <number> --owner <owner> --format json

# Get project ID (needed for item-edit)
gh project view <number> --owner <owner> --format json --jq '.id'

# Open in browser
gh project view <number> --owner <owner> --web
```

## Edit

```bash
# Update title
gh project edit <number> --owner <owner> --title "New Title"

# Update description and visibility
gh project edit <number> --owner <owner> --description "Cross-repo roadmap" --visibility PRIVATE
```

## Close / Reopen

```bash
# Close a completed project
gh project close <number> --owner <owner>

# Reopen
gh project close <number> --owner <owner> --undo
```

## Delete

```bash
# Permanently delete (no undo)
gh project delete <number> --owner <owner>
```

## Repository Linking

Link repositories to make their issues available in the project.

### Link

```bash
# Link a specific repo
gh project link <number> --owner <owner> --repo <owner>/<repo>

# Link the current directory's repo
gh project link <number> --owner <owner>

# Link a team
gh project link <number> --owner <org> --team <team-slug>
```

### Unlink

```bash
# Unlink a repo
gh project unlink <number> --owner <owner> --repo <owner>/<repo>
```

## Bootstrap Labels

Replace the default GitHub labels with the standard TAU Platform set:

```bash
# Remove default labels
for label in "bug" "documentation" "duplicate" "enhancement" "good first issue" \
  "help wanted" "invalid" "question" "wontfix"; do
  gh label delete "$label" --repo <owner>/<repo> --yes 2>/dev/null
done

# Create standard labels
gh label create "bug"             --repo <owner>/<repo> --color d73a4a --description "Something isn't working correctly"
gh label create "feature"         --repo <owner>/<repo> --color 0075ca --description "New capability or functionality"
gh label create "improvement"     --repo <owner>/<repo> --color a2eeef --description "Enhancement to existing functionality"
gh label create "refactor"        --repo <owner>/<repo> --color d4c5f9 --description "Code restructuring without behavior change"
gh label create "documentation"   --repo <owner>/<repo> --color 0e8a16 --description "Documentation additions or updates"
gh label create "testing"         --repo <owner>/<repo> --color fbca04 --description "Test additions or improvements"
gh label create "infrastructure"  --repo <owner>/<repo> --color e4e669 --description "CI/CD, build, tooling, project setup"
```

### Clone Labels Across Repos

Once a source repo has the standard labels, clone them to new repos:

```bash
# Clone labels from source repo (overwrites existing)
gh label clone <owner>/tau-platform --repo <owner>/<new-repo> --force
```

## Milestone Convention

Each non-meta phase gets a corresponding milestone on every linked repository.

### Create Milestones

```bash
# Create milestone on each linked repo
gh api --method POST /repos/<owner>/<repo>/milestones \
  -f title="Phase 1 - Foundation" \
  -f description="Foundation phase -- see TAU Platform project board"
```

### Assign Milestone When Assigning Phase

```bash
gh issue edit <number> --repo <owner>/<repo> --milestone "Phase 1 - Foundation"
```
