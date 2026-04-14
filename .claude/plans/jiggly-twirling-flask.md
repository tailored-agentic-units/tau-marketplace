# Remove "Set Issue Status" from iterative-dev init

## Context

The `iterative-dev` skill is project-agnostic and does not integrate with GitHub Projects. Its current `init` command tells the AI to run `gh issue edit <number> --add-label "in-progress"` at session start, which is wrong for both workflows it could plausibly serve:

- **Projects-backed repos** — issue status is a project board field (`gh project item-edit ...`), not a label on the issue.
- **Plain repos** (the skill's default) — there is no "status" concept at all; the label call just pollutes the repo or fails if the label doesn't exist.

Fix: strip the step. Branch creation already signals "session in flight"; the closing PR signals "done." Release the fix as `iterative-dev` v0.1.2.

Full rationale is captured in `.claude/remove-issue-status.md`.

## Changes

### 1. `plugins/iterative-dev/skills/iterative-dev/commands/init.md`

Remove lines 17–22 (the entire **Set Issue Status** subsection):

```markdown
### Set Issue Status

```bash
gh issue edit <number> --add-label "in-progress"
```
```

Resulting Stage 1 flow: Read the Issue → Create a Session Branch → Read Project Context.

No content replacement needed.

### 2. `plugins/iterative-dev/.claude-plugin/plugin.json`

Bump `"version"` from `"0.1.1"` to `"0.1.2"`.

### 3. `plugins/iterative-dev/CHANGELOG.md`

Insert a new section directly below the `# Changelog` heading and above `## [v0.1.1]`:

```markdown
## [v0.1.2]
- Remove "Set Issue Status" step from init command — issue labels are the wrong mechanism for session status in a project-agnostic skill (commands/init.md)
```

## Workflow

Per the `tau-marketplace-dev` update session flow:

1. `git checkout -b update/remove-issue-status`
2. Apply the three edits above.
3. `git add .`
4. `git commit -m "Remove Set Issue Status step from iterative-dev init"`
5. `git push -u origin update/remove-issue-status`
6. `gh pr create` with a summary referencing `.claude/remove-issue-status.md` and the v0.1.1 → v0.1.2 bump.

After merge (handled as a separate release session): tag `iterative-dev/v0.1.2` is pushed to trigger the release workflow.

## Verification

- `cat plugins/iterative-dev/skills/iterative-dev/commands/init.md` — confirm no `### Set Issue Status` heading and no `gh issue edit ... in-progress` invocation remain; Stage 1 reads as the three-step flow.
- `jq '.version' plugins/iterative-dev/.claude-plugin/plugin.json` → `"0.1.2"`.
- `head -20 plugins/iterative-dev/CHANGELOG.md` — confirm `## [v0.1.2]` sits at the top with the bullet.
- `gh pr view` — PR is open against `main` with the expected title/body.

## Critical Files

- `plugins/iterative-dev/skills/iterative-dev/commands/init.md`
- `plugins/iterative-dev/.claude-plugin/plugin.json`
- `plugins/iterative-dev/CHANGELOG.md`
- `.claude/remove-issue-status.md` (source of truth for the change; not modified)
