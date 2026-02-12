# Update Session

## Purpose

Initialize a structured update session for modifying the plugin ecosystem. Handles branch creation, planning, execution, version bumps, CHANGELOG entries, and PR creation.

## Workflow

### Step 1: Receive Update Description

Collect from the user:
- Description of changes to make
- Which plugin(s) are affected (default: `tau`)
- Version bump type: patch (default), minor, or major

### Step 2: Create Branch

Create a branch named for the changes:

```bash
git checkout -b update/<brief-slug>
```

Branch name should be a short kebab-case description of the update (e.g., `update/refine-dev-workflow-commands`).

### Step 3: Plan Updates

Explore the affected files and plan the changes:

- Identify all files that need modification
- Draft the scope of changes for each file
- Consider cross-file consistency (e.g., SKILL.md descriptions matching command content)
- Present the plan for user approval

### Step 4: Execute Updates

Implement the planned changes across all affected files.

### Step 5: Version Bump

Bump the version in `plugins/[plugin]/.claude-plugin/plugin.json` for each affected plugin:

- **Patch** (default): bug fixes, documentation updates, minor refinements
- **Minor**: new skills, new sub-commands, significant feature additions
- **Major**: breaking changes to skill interfaces or conventions

### Step 6: CHANGELOG Entry

Add a new versioned section at the top of `CHANGELOG.md` (below the `# Changelog` heading):

```markdown
## [new-version]

- [Description of change 1]
- [Description of change 2]
```

The heading uses bare semver (e.g., `## 0.0.8`) — no `v` prefix, no plugin prefix. This format is required for compatibility with the release workflow.

### Step 7: Commit, Push, and PR

Commit all changes, push the branch, and open a PR:

```bash
git add .
git commit -m "<descriptive message>"
git push -u origin update/<brief-slug>
```

Create the PR:

```bash
gh pr create --title "<concise summary>" --body "$(cat <<'EOF'
## Summary

[What was updated and why]

## Changes

- [Key changes by file or area]

## Version

- [plugin]: [old-version] → [new-version]
EOF
)"
```

## Outcomes

- All planned updates are implemented
- Plugin version(s) bumped in plugin.json
- CHANGELOG entry added for the new version
- Branch pushed and PR opened for review
