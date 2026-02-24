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
├── CHANGELOG.md                   # Release notes (heading format: ## [version])
├── plugins/
│   └── tau/
│       ├── .claude-plugin/
│       │   └── plugin.json        # Version source of truth
│       └── skills/                # Plugin skills
└── README.md
```

## Versioning Convention

- Each plugin has a version in `plugins/[plugin]/.claude-plugin/plugin.json`
- CHANGELOG headings use bare semver: `## 0.0.8` (no `v` prefix, no plugin prefix)
- Tags use the format: `[plugin]-v[major].[minor].[patch]` (e.g., `tau-v0.0.8`)
- The release workflow extracts the plugin prefix from the tag and matches it to the CHANGELOG heading

## Session Types

### Update Session

Initializes a structured update session for modifying the plugin ecosystem.

See [update.md](./commands/update.md) for the full workflow.

### Release Session

After a PR from an update session is merged, creates and pushes release tags.

See [release.md](./commands/release.md) for the full workflow.
