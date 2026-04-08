---
name: tau-overview
description: >
  TAU ecosystem plugin overview and conventions. Use for any TAU project
  development activity. Triggers: tau, development session, context document,
  session continuity, plan file, skill catalog.
user-invocable: false
---

# TAU Marketplace Overview

The TAU marketplace provides standalone plugins for the Tailored Agentic Units ecosystem. Each plugin is independently installable, versioned, and focused on a single concern.

## Skills

Skills that are installed load automatically based on context:

| Skill | Use When |
|-------|----------|
| dev-workflow:dev-workflow | Development sessions: concept development (vision + phases), planning (phase → objectives, objective → tasks), task execution, project review, release |
| github-cli:github-cli | GitHub repository operations via gh CLI (issues, PRs, releases, labels, secrets, sub-issues) |
| go-patterns:go-patterns | Go design patterns: interfaces, error handling, package structure, configuration |
| project-management:project-management | GitHub Projects v2: project boards, phases, objectives, cross-repo backlog management |


### Cross-Skill Integration

The `dev-workflow:dev-workflow` skill orchestrates structured development sessions that load other skills as needed:

| Session Type | Skills Loaded |
|-------------|---------------|
| Concept Development | project-management:project-management, github-cli:github-cli |
| Planning | project-management:project-management, github-cli:github-cli |
| Task Execution | github-cli:github-cli |
| Project Review | project-management:project-management, github-cli:github-cli |
| Release | github-cli:github-cli |

## Context Documents

Project knowledge artifacts stored in `.claude/context/`:

| Directory | Contents | Naming |
|-----------|----------|--------|
| `concepts/` | Architectural concept documents | `[slug].md` |
| `guides/` | Active implementation guides | `[issue-number]-[slug].md` |
| `sessions/` | Session summaries | `[issue-number]-[slug].md` |
| `reviews/` | Project review reports | `[YYYY-MM-DD]-[scope].md` |

Each directory has a `.archive/` subdirectory for completed documents. Directories are created on demand. See the `dev-workflow:dev-workflow` skill for lifecycle and conventions.

## Session Continuity

Plan files in `.claude/plans/` enable session continuity across machines.

### Saving Session State

When pausing work, append a context snapshot to the active plan file:

```markdown
## Context Snapshot - [YYYY-MM-DD HH:MM]

**Current State**: [Brief description of where work stands]

**Files Modified**:
- [List of files changed this session]

**Next Steps**:
- [Immediate next action]
- [Subsequent actions]

**Key Decisions**:
- [Important decisions made and rationale]

**Blockers/Questions**:
- [Any unresolved issues]
```

### Restoring Session State

When continuing work from a plan file:

1. Read the plan file to restore context
2. Review the most recent Context Snapshot
3. Resume from the documented Next Steps
4. Update the snapshot when pausing again
