---
name: tau-overview
description: >
  TAU ecosystem plugin overview and conventions. Use for any TAU project
  development activity. Triggers: tau, development session, context document,
  session continuity, plan file, skill catalog.
user-invocable: false
---

# TAU Plugin Overview

The tau plugin provides shared skills, conventions, and tooling for the Tailored Agentic Units ecosystem.

## Skills

Skills load automatically based on context:

| Skill | Use When |
|-------|----------|
| tau:dev-workflow | Development sessions: concept development, task execution, project review, release |
| tau:github-cli | GitHub repository operations via gh CLI (issues, PRs, releases, labels, secrets) |
| tau:go-patterns | Go design patterns: interfaces, error handling, package structure, configuration |
| tau:project-management | GitHub Projects v2: project boards, phases, cross-repo backlog management |
| tau:skill-creator | Creating or modifying Claude Code skills |
| tau:tau-core | Building applications with tau-core: agents, protocols, providers, configuration |
| tau:tau-orchestrate | Building applications with tau-orchestrate: hub coordination, state, workflows |

### Cross-Skill Integration

The `tau:dev-workflow` skill orchestrates structured development sessions that load other skills as needed:

| Session Type | Skills Loaded |
|-------------|---------------|
| Concept Development | tau:project-management, tau:github-cli |
| Task Execution | tau:github-cli, tau:go-patterns, tau:skill-creator (+ dev-type references) |
| Project Review | tau:project-management, tau:github-cli |
| Release | tau:github-cli (+ dev-type references) |

## Context Documents

Project knowledge artifacts stored in `.claude/context/`:

| Directory | Contents | Naming |
|-----------|----------|--------|
| `concepts/` | Architectural concept documents | `[slug].md` |
| `guides/` | Active implementation guides | `[issue-number]-[slug].md` |
| `sessions/` | Session summaries | `[issue-number]-[slug].md` |
| `reviews/` | Project review reports | `[YYYY-MM-DD]-[scope].md` |

Each directory has a `.archive/` subdirectory for completed documents. Directories are created on demand. See the `tau:dev-workflow` skill for lifecycle and conventions.

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
