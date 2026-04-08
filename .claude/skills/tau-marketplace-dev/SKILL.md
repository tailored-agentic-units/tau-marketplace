---
name: tau-marketplace-dev
argument-hint: "[update | release]"
description: >
  REQUIRED for tau-marketplace development and maintenance sessions.
  Use when the user asks to "update skills", "update plugin", "make changes",
  "release", "tag release", "bump version", or any marketplace maintenance activity.
  Triggers: update, release, bump version, marketplace changes, skill update,
  plugin update, tag, version bump, update session.

  When this skill is invoked, identify the session type from the argument
  (update or release) and follow the corresponding workflow in commands/.
user-invocable: true
---

# TAU Marketplace Development

## When This Skill MUST Be Used

**ALWAYS invoke this skill when the user's request involves ANY of these:**

- Updating or modifying plugin skills in the marketplace
- Bumping plugin versions
- Creating releases or tags for the marketplace
- Any structured marketplace maintenance session

## Repository Structure

```
tau-marketplace/
├── .claude/
│   ├── settings.json
│   └── skills/
│       └── tau-marketplace-dev/   # This skill
├── .github/
│   └── workflows/
│       └── release.yml            # Tag-triggered release automation
├── plugins/
│   ├── dev-workflow/
│   │   ├── .claude-plugin/plugin.json
│   │   ├── CHANGELOG.md
│   │   └── skills/dev-workflow/
│   ├── github-cli/
│   │   ├── .claude-plugin/plugin.json
│   │   ├── CHANGELOG.md
│   │   └── skills/github-cli/
│   ├── go-patterns/
│   │   ├── .claude-plugin/plugin.json
│   │   ├── CHANGELOG.md
│   │   └── skills/go-patterns/
│   ├── iterative-dev/
│   │   ├── .claude-plugin/plugin.json
│   │   ├── CHANGELOG.md
│   │   └── skills/iterative-dev/
│   ├── project-management/
│   │   ├── .claude-plugin/plugin.json
│   │   ├── CHANGELOG.md
│   │   └── skills/project-management/


└── README.md
```

## Versioning Convention

- Each plugin has a version in `plugins/[plugin]/.claude-plugin/plugin.json`
- Each plugin has its own `CHANGELOG.md` at `plugins/[plugin]/CHANGELOG.md`
- CHANGELOG headings use bracketed semver: `## [v0.1.0]` (required by `parse-changelog`)
- Tags use the format: `[plugin]/v[major].[minor].[patch]` (e.g., `dev-workflow/v0.1.0`)
- The release workflow extracts the plugin prefix from the tag and reads the corresponding plugin's CHANGELOG

## Session Types

### Update Session

Initializes a structured update session for modifying the plugin ecosystem.

See [update.md](./commands/update.md) for the full workflow.

### Release Session

After a PR from an update session is merged, creates and pushes release tags.

See [release.md](./commands/release.md) for the full workflow.
