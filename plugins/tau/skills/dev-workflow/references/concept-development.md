# Concept Development Session

## Purpose

Develop a new idea or capability from rough concept to actionable plan with full project-management infrastructure. The outcome is either an extension of an existing repository with new features or the initialization of an entirely new project.

## Prerequisites

- Load skills: **tau:project-management**, **tau:github-cli**
- Collect session metadata: concept name, scope, target repositories

## Workflow

### Phase 1: Explore and Understand

- Read existing codebase and architecture relevant to the concept
- Review existing concept documents in `.claude/context/concepts/`
- Review current project-management state (projects, phases, backlog)
- Identify patterns established in prior work that the concept should align with

### Phase 2: Concept Definition

- Articulate the problem space and solution approach
- Define boundaries: what is in scope, what is not
- Identify architectural constraints and trade-offs
- Document dependencies (libraries, services, infrastructure)
- Estimate complexity and break into discrete work items

### Phase 3: Collaborative Discussion

- Surface strategic questions and alternatives
- Discuss trade-offs with rationale
- Resolve through collaborative iteration, not assumptions
- Document key decisions and the reasoning behind them

### Phase 4: Infrastructure Creation

Determine whether the concept extends an existing repository or requires a new project.

**If initializing a new project (do first if applicable):**

1. Create the repository
2. Bootstrap standard labels (see tau:project-management skill: Label Convention)
3. Link repository to the project board
4. Create the Phase field if it doesn't exist, or add new phase options
5. Create milestones matching the phases on linked repositories

**Create Objectives with sub-issues:**

1. Create an **Objective** (parent issue) on the primary repository for each cohesive set of requirements
2. Create **sub-issues** on their respective repositories with bootstrap context in the body
3. Link sub-issues to their Objective via the GraphQL API (see tau:project-management skill: Objective Convention)
4. Apply standard labels from the label taxonomy
5. Add Objectives and sub-issues to the project board
6. Assign to the appropriate phase and corresponding milestones

**Issue body convention:**

Each issue body should contain enough context to bootstrap a task execution session:

```markdown
## Context

[Why this work is needed and how it fits into the concept]

## Scope

[What this issue covers and what it does not]

## Approach

[Recommended implementation strategy, if known]

## Dependencies

[Issues or components this depends on]

## Acceptance Criteria

- [ ] [Specific, verifiable criteria]
```

### Phase 5: Concept Documentation

Create a concept document at `.claude/context/concepts/[slug].md`.

Ensure the directory exists before writing:

```bash
mkdir -p .claude/context/concepts
```

**Concept document template:**

```markdown
# [Concept Name]

## Core Premise

[What this concept is and why it exists]

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

## Related Issues

- [owner/repo#number] - [issue title]
```

## Outcomes

At the end of a concept development session:

- Concept document exists in `.claude/context/concepts/`
- Objectives created as parent issues with the `objective` label
- Sub-issues created on respective repositories with bootstrap context in bodies
- Sub-issues linked to their Objectives via GraphQL API
- Issues added to the project board with phase assignments
- Milestones created on affected repositories
- Labels applied from standard taxonomy
