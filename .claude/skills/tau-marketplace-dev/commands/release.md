# Release Session

## Purpose

After a PR from an update session is merged, create and push release tags that trigger the GitHub Actions release workflow.

## Prerequisites

- The update PR has been merged to `main`
- Working tree is clean and on `main` with latest changes pulled

## Workflow

### Step 1: Identify Affected Plugins

Read the merged PR to identify which plugin(s) were updated and their new versions:

```bash
# View the most recently merged PR
gh pr list --state merged --limit 1 --json number,title,body
```

Alternatively, read `plugins/[plugin]/.claude-plugin/plugin.json` directly to get the current version.

### Step 2: Verify CHANGELOG

For each affected plugin, verify that `CHANGELOG.md` has a heading matching the new version:

```
## [version]
```

For example, if the plugin version is `0.0.8`, verify that `## 0.0.8` exists in the CHANGELOG.

### Step 3: Create and Push Tags

For each affected plugin, create and push a tag:

```bash
# Tag format: [plugin]-v[version]
# Example: tau-v0.0.8
git tag tau-v0.0.8
git push origin tau-v0.0.8
```

If multiple plugins are affected, create one tag per plugin:

```bash
git tag tau-v0.0.8
git tag other-plugin-v1.0.0
git push origin tau-v0.0.8
git push origin other-plugin-v1.0.0
```

### Step 4: Verify Release

The tag push triggers `.github/workflows/release.yml`, which:

1. Extracts the plugin prefix from the tag (e.g., `tau` from `tau-v0.0.8`)
2. Uses `parse-changelog` to extract the matching section from `CHANGELOG.md`
3. Creates a GitHub Release with the extracted notes

Verify the release was created:

```bash
gh release view tau-v0.0.8
```

## Outcomes

- Release tag(s) created and pushed
- GitHub Actions workflow triggered
- GitHub Release(s) created with CHANGELOG notes
