# Update Plan: Remove skill-creator, add release branch management, expand phase closeout

## Context

Three updates requested in `updates.md`:
1. The `skill-creator` skill duplicates capability now provided by Anthropic's official plugin — remove it and all references
2. Release sessions assume the working tree is on `main`, but they always follow a merged PR where the branch may be stale — add a Step 0 to handle branch cleanup
3. The phase closeout (Step 0 in `phase.md`) only covers objective disposition and document cleanup, but lacks the full release lifecycle (milestone closing, CHANGELOG consolidation, dev release/tag cleanup, phase release tagging) — these were done manually during Herald Phase 2 and need to be standardized

Current version: `0.1.1` → bump to `0.1.2` (patch)

## Changes

### 1. Delete `skills/skill-creator/`

Delete the entire directory:
- `plugins/tau/skills/skill-creator/SKILL.md`
- `plugins/tau/skills/skill-creator/references/frontmatter-reference.md`

### 2. Remove skill-creator references

**`plugins/tau/README.md`:**
- Delete the Skill Authoring section (lines 38–42: heading + table)
- Update Cross-Skill Integration table row (line 52): remove `, skill-creator` → `| Task Execution | github-cli, go-patterns, plus dev-type references |`

**`plugins/tau/skills/tau-overview/SKILL.md`:**
- Delete the skill-creator row from the skills table (line 24)
- Update Cross-Skill Integration table row (line 35): remove `, tau:skill-creator` → `| Task Execution | tau:github-cli, tau:go-patterns (+ dev-type references) |`

**`README.md` (root):**
- Remove `"Skill(tau:skill-creator)"` from the permission example (line 47)
- Remove `"Skill(tau:skill-creator)"` from the complete example (line 96)
- Update skill count in repo structure comment (line 133): `7 skills` → `6 skills`

### 3. Add Step 0: Sync Main to `release.md`

**File:** `plugins/tau/skills/dev-workflow/commands/release.md`

**Prerequisites (lines 15–16):** Remove the two bullets about clean working tree and current branch being main.

**Dev Release — insert before Step 1 (after line 44):**
```markdown
### Step 0: Sync Main

Releases always follow a merged PR. Sync main and clean up the stale task branch:

\```bash
BRANCH=$(git branch --show-current)

git checkout main
git pull origin main

# Delete the stale task branch and prune remote tracking refs
git branch -d "$BRANCH"
git remote prune origin
\```
```

**Phase Release — insert before Step 1: Validate (after line 81):**
Same Step 0: Sync Main content (worded slightly differently per `updates.md`).

No renumbering needed — existing steps start at 1.

### 4. Expand phase closeout in `phase.md`

**File:** `plugins/tau/skills/dev-workflow/commands/phase.md`

Keep 0a (Status Assessment, lines 48–57) and 0b (Disposition of Incomplete Objectives, lines 59–86) as-is.

Replace 0c (Clean Slate, lines 88–91) with four new sub-steps:

**0c. Close Milestone** — Close the milestone associated with the phase version target via `gh api`.

**0d. Update Project Documents** — Replaces old 0c. Clear `_project/phase.md` and `_project/objective.md` to empty placeholders. Update `_project/README.md`: add Status column to Phases table, mark completed phase as "Complete", fix stale descriptions.

**0e. Consolidate CHANGELOG and Clean Up Dev Releases** — Four sub-steps:
1. Consolidate CHANGELOG: replace all `## v<target>-dev.*` sections with single `## v<target>` section
2. Delete dev releases via `gh release delete`
3. Delete container registry images (if applicable) via `gh api`
4. Delete dev tags locally and on remote

**0f. Commit, PR, and Tag Phase Release** — Three sub-steps:
1. Create branch, commit, push closeout changes
2. Create PR for the closeout
3. After PR merges, sync main and tag the phase release

Move the `---` separator from after old 0c (line 93) to after new 0f.

**Outcomes section (line 193):** Add bullet:
`- Previous phase released (if transitioning): milestone closed, CHANGELOG consolidated, dev releases and tags cleaned up, phase version tagged`

### 5. Version bump and CHANGELOG

**`plugins/tau/.claude-plugin/plugin.json`:** `0.1.1` → `0.1.2`

**`CHANGELOG.md`:** Add new section at top:
```markdown
## 0.1.2

- Remove skill-creator skill — capability now provided by Anthropic's official plugin
- Add Step 0 (Sync Main) to dev and phase release workflows for automatic branch cleanup after merged PRs
- Expand phase closeout from 3 sub-steps to 6: add milestone closing, CHANGELOG consolidation, dev release/tag/image cleanup, and phase release tagging
```

## Files Changed

| File | Action |
|------|--------|
| `plugins/tau/skills/skill-creator/SKILL.md` | Delete |
| `plugins/tau/skills/skill-creator/references/frontmatter-reference.md` | Delete |
| `plugins/tau/README.md` | Remove Skill Authoring section, update Cross-Skill Integration table |
| `plugins/tau/skills/tau-overview/SKILL.md` | Remove skill-creator from skills table and integration table |
| `README.md` | Remove skill-creator from permission examples, update skill count |
| `plugins/tau/skills/dev-workflow/commands/release.md` | Add Step 0: Sync Main to both release types, remove stale prerequisites |
| `plugins/tau/skills/dev-workflow/commands/phase.md` | Expand Step 0 from 3 to 6 sub-steps (0a–0f) |
| `plugins/tau/.claude-plugin/plugin.json` | Version bump 0.1.1 → 0.1.2 |
| `CHANGELOG.md` | Add 0.1.2 entry |

## Verification

1. Confirm no remaining references to `skill-creator` in the repo (grep)
2. Validate JSON syntax in `plugin.json`
3. Review CHANGELOG heading format matches release workflow expectations (bare semver, no `v` prefix)
4. Read through modified files end-to-end for consistency
