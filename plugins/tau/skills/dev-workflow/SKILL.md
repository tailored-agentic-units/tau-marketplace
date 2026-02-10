---
name: dev-workflow
argument-hint: "[concept | plan phase | plan objective <issue-number> | task <issue-number> | review | release <version>]"
description: >
  REQUIRED for structured development workflow sessions.
  Use when the user asks to "start a session", "concept development",
  "plan phase", "plan objective", "task execution", "project review",
  "start development", "create concept", "review project", "session closeout",
  "tag a release", "create release", or any structured development activity.
  Triggers: session, concept, plan, planning, phase planning, objective planning,
  task execution, project review, implementation guide, closeout, development workflow,
  start session, review session, concept session, release, tag, version.

  When this skill is invoked, identify the session type from the argument
  (concept, plan, task, review, or release) and follow the corresponding workflow
  in the commands/ directory. Load the tau:project-management skill for concept
  development, planning, and project review sessions. Load domain-specific skills
  for task execution and release sessions.
---

# Development Workflow

## When This Skill MUST Be Used

**ALWAYS invoke this skill when the user's request involves ANY of these:**

- Starting any structured development session (concept, plan, task, review, or release)
- Creating or developing a new concept
- Planning a phase (breaking it into objectives) or an objective (breaking it into sub-issues)
- Executing a development task from a GitHub issue
- Reviewing project infrastructure and codebase health
- Tagging a release
- Session closeout activities
- Creating or archiving context documents

## Session Metadata

Before starting a session, collect the required metadata based on session type.

**Invocation patterns:**

| Pattern | Session Type | Example |
|---------|-------------|---------|
| `/dev-workflow concept` | Concept Development | `/dev-workflow concept` |
| `/dev-workflow plan phase` | Phase Planning | `/dev-workflow plan phase` |
| `/dev-workflow plan objective <issue>` | Objective Planning | `/dev-workflow plan objective 5` |
| `/dev-workflow task <issue-number>` | Task Execution | `/dev-workflow task 42` |
| `/dev-workflow review` | Project Review | `/dev-workflow review` |
| `/dev-workflow release <version>` | Release | `/dev-workflow release v0.2.0` |

### Concept Development

1. **Concept name** - descriptive slug (e.g., `audio-protocol`). Derived from argument or collected from user.
2. **Scope** - extend existing repository or initialize new project.
3. **Target repositories** - which repos are affected by the concept.

### Phase Planning

1. **Phase name** - identified from the project board or collected from the user.
2. **Primary repository** - where Objectives will be created.
3. **`_project/README.md`** - read from the primary repository for vision and architecture context.

### Objective Planning

1. **Issue number** - required, from argument. The Objective (parent issue) to plan.
2. **Target repositories** - identified from the Objective body or collected from the user.

### Task Execution

1. **Issue number** - required, from argument. Read the issue to bootstrap session context.
2. **Development type** - auto-detected from issue labels or collected. Determines which dev-type reference to load from `dev-types/`.

### Project Review

1. **Scope** - repository name or project name being reviewed. Defaults to current repository.

### Release

1. **Version** - required, from argument (e.g., `/dev-workflow release v0.2.0`). Must follow semantic versioning.

## Session Types

### Concept Development Session

Develops a new idea from rough concept to actionable plan with full project-management infrastructure. Produces a concept document and may establish new phases, repositories, or project structure.

**Skills loaded:** tau:project-management, tau:github-cli

See [concept-development.md](commands/concept-development.md) for the full workflow.

### Planning Session

Breaks down project structure into actionable units. Two variants:

**Phase Planning** — Takes a phase and produces Objectives (parent issues with the `objective` label). Each Objective represents a cohesive set of requirements within the phase. Objectives are created on the primary repository.

**Objective Planning** — Takes an Objective and produces sub-issues across the relevant repositories. This is where architecture details are ironed out and the scope of each development session is defined. Each sub-issue maps to exactly one branch and one PR.

**Skills loaded:** tau:project-management, tau:github-cli

See [planning.md](commands/planning.md) for the full workflow.

### Task Execution Session

Issue-driven development session starting in Claude Code plan mode. Creates a branch, collaboratively plans the implementation, generates an implementation guide, and performs structured closeout including PR creation and project-management updates.

**Skills loaded:** tau:github-cli, plus domain-specific skills based on development type

See [task-execution.md](commands/task-execution.md) for the full workflow.

### Project Review Session

Evaluates the alignment between project-management infrastructure and the current state of the codebase. Ensures the long-term vision is on track and produces a review report.

**Skills loaded:** tau:project-management, tau:github-cli

See [project-review.md](commands/project-review.md) for the full workflow.

### Release Session

Two release types: **dev releases** (per-PR pre-release tags like `v0.1.0-dev.03`) and **phase releases** (final version like `v0.1.0`). Dev releases are triggered as part of task execution closeout. Phase releases finalize a version by converting CHANGELOG entries, running validation, and tagging all phase modules. Dev-type references augment with tag format conventions (e.g., Go modules require a `v` prefix).

**Skills loaded:** tau:github-cli, plus domain-specific skills based on development type

See [release.md](commands/release.md) for the full workflow.

## Context Document Convention

Context documents are persistent project knowledge artifacts stored in `.claude/context/`. They are distinct from plan files (`.claude/plans/`), which are ephemeral session working memory.

### Directory Structure

```
.claude/context/
├── concepts/                         # Concept development outputs
│   └── .archive/                     # Superseded or completed concepts
├── guides/                           # Active implementation guides
│   └── .archive/                     # Archived after session closeout
├── sessions/                         # Session summaries
│   └── .archive/                     # Historical summaries
└── reviews/                          # Project review reports
    └── .archive/                     # Historical reviews
```

Directories are created on demand when the first artifact is placed in them.

### Naming Conventions

| Type | Pattern | Example |
|------|---------|---------|
| Concept | `[slug].md` | `audio-protocol.md` |
| Implementation guide | `[issue-number]-[slug].md` | `42-audio-protocol.md` |
| Session summary | `[issue-number]-[slug].md` | `42-audio-protocol.md` |
| Review report | `[YYYY-MM-DD]-[scope].md` | `2026-02-03-tau-core.md` |

### Lifecycle

| State | Location | Trigger |
|-------|----------|---------|
| Active concept | `context/concepts/[slug].md` | Created during concept development |
| Archived concept | `context/concepts/.archive/[slug].md` | Concept fully implemented or superseded |
| Active guide | `context/guides/[issue-number]-[slug].md` | Created during task execution |
| Archived guide | `context/guides/.archive/[issue-number]-[slug].md` | Moved during session closeout |
| Session summary | `context/sessions/[issue-number]-[slug].md` | Created during session closeout |
| Archived summary | `context/sessions/.archive/[issue-number]-[slug].md` | Moved during project review when no longer referenced |
| Active review | `context/reviews/[YYYY-MM-DD]-[scope].md` | Created during project review |
| Archived review | `context/reviews/.archive/[YYYY-MM-DD]-[scope].md` | Superseded by a newer review |

### Relationship to Plan Files

| Aspect | Plan Files (`.claude/plans/`) | Context Documents (`.claude/context/`) |
|--------|-------------------------------|---------------------------------------|
| Purpose | Session-scoped working memory | Persistent project knowledge |
| Created by | Claude Code plan mode | Session workflows |
| Lifecycle | Disposable after session | Versioned, archived, never deleted |
| Naming | Auto-generated slugs | Structured conventions |

A task execution session starts in plan mode (producing a plan file), then transforms the plan into an implementation guide in `.claude/context/guides/`. The plan file can be cleaned up at the developer's discretion.

## Skill Integration

| Session Type | Always Loaded | Conditionally Loaded |
|-------------|---------------|---------------------|
| Concept Development | tau:project-management, tau:github-cli | Domain skills based on concept scope |
| Planning | tau:project-management, tau:github-cli | Domain skills for architecture context |
| Task Execution | tau:github-cli | Dev-type skills (e.g., tau:go-patterns), domain skills based on issue labels |
| Project Review | tau:project-management, tau:github-cli | Domain skills for codebase assessment |
| Release | tau:github-cli | Dev-type skills for tag format and validation |

> **Scope boundary**: This skill orchestrates session workflows. It delegates specific operations to the appropriate skill (tau:project-management for infrastructure, tau:github-cli for issue/PR operations, domain skills for implementation patterns).
