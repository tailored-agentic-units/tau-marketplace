# Concept Development Session

## Purpose

Develop a new idea or capability from initial requirements to a project concept with long-term vision. The outcome is a defined project with Phases mapped out as areas of focus, repository infrastructure initialized, and a GitHub Project board ready for phase planning.

## Prerequisites

- Load skills: **tau:project-management**, **tau:github-cli**
- Collect session metadata: concept name, scope, target repositories

## Workflow

### Phase 1: Explore and Understand

- Receive and review the initial set of ideas and requirements from the user
- Read existing codebase and architecture relevant to the concept
- Review existing concept documents in `.claude/context/concepts/`
- Review current project-management state (projects, phases, backlog)
- Identify patterns established in prior work that the concept should align with

### Phase 2: Concept Definition

- Articulate the problem space and solution approach
- Define boundaries: what is in scope, what is not
- Identify architectural constraints and trade-offs
- Document dependencies (libraries, services, infrastructure)
- Develop the long-term vision for what the project will deliver
- Map out the initial set of Phases — each Phase is an area of focus, not a detailed implementation plan

### Phase 3: Collaborative Discussion

- Surface strategic questions and alternatives
- Discuss trade-offs with rationale
- Refine Phase boundaries and ordering
- Resolve through collaborative iteration, not assumptions
- Document key decisions and the reasoning behind them

### Phase 4: Infrastructure Creation

Determine whether the concept extends an existing repository or requires a new project.

**If initializing a new project (do first if applicable):**

1. Create the repository at the specified local and remote location
2. Bootstrap standard labels (see tau:project-management skill: Label Convention)
3. Link repository to the project board

**Configure project board phases:**

1. Create the Phase field if it doesn't exist
2. Add a phase option for each identified Phase
3. Create milestones matching the phases on each linked repository

### Phase 5: Concept Documentation

The documentation output depends on whether the concept initializes a new project.

**If the concept initializes a new project:**

The concept document becomes `_project/README.md` in the target repository — the project identity
and implementation context document. The vision section leads, followed by phases, architecture,
conventions, and any other project-specific details relevant to development sessions.

Ensure the directory exists:

```bash
mkdir -p _project
```

**If the concept is exploratory / not ready for project initialization:**

Create a concept document at `.claude/context/concepts/[slug].md`.

Ensure the directory exists before writing:

```bash
mkdir -p .claude/context/concepts
```

**Concept document template:**

```markdown
# [Concept Name]

## Vision

[Long-term vision for what this project will deliver]

## Core Premise

[What this concept is and why it exists]

## Phases

| Phase | Focus Area | Version Target |
|-------|-----------|----------------|
| Phase 1 - [Name] | [Area of focus for this phase] | v[major].[minor].[patch] |
| Phase 2 - [Name] | [Area of focus for this phase] | v[major].[minor].[patch] |

## Architecture

[High-level architecture, component relationships, data flow]

## Key Decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| [Decision point] | [What was chosen] | [Why] |

## Dependencies

- [Libraries, services, or infrastructure required]

## Integration Points

- [How this concept connects to existing systems]

## Open Questions

- [Unresolved questions for future sessions]
```

## Outcomes

At the end of a concept development session:

- **If project initialized**: `_project/README.md` exists as the authoritative project document
- **If exploratory**: Concept document exists in `.claude/context/concepts/`
- Phases defined with focus areas and version targets
- Phase field options created on the project board for each Phase
- Milestones created on affected repositories matching Phases
- Repository linked to the project board (if new project)
