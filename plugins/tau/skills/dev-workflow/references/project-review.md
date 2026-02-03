# Project Review Session

## Purpose

Evaluate the alignment between project-management infrastructure and the current state of the codebase. Ensure the long-term vision is on track, validate that sufficient project-management resources exist to continue progress, and confirm that code, testing, documentation, and context infrastructure are in a well-structured, accurate state.

## Prerequisites

- Load skills: **tau:project-management**, **tau:github-cli**
- Collect session metadata: scope (defaults to current repository)

## Workflow

### Phase 1: Infrastructure Audit

Evaluate the project-management infrastructure:

- List projects and phases using the tau:project-management skill
- List open issues across linked repositories
- Verify label taxonomy consistency across repos (compare against standard labels)
- Check milestone alignment with phases (each non-meta phase should have a corresponding milestone on each linked repo)
- Identify orphaned issues (open issues not on the project board)
- Identify stale issues (open issues with no recent activity)

### Phase 2: Codebase Assessment

Evaluate the health of the codebase:

- **Code quality**: Architectural consistency, adherence to established patterns
- **Test coverage**: Run coverage and evaluate critical path coverage
- **Documentation**: Completeness of code documentation (exported types, functions, methods)
- **Dependencies**: Outdated or unused dependencies
- **Technical debt**: Areas where shortcuts were taken or patterns diverged

### Phase 3: Context Infrastructure Review

Audit the `.claude/context/` directory:

- Identify concept documents that should be archived (fully implemented or superseded)
- Check that session summaries exist for completed work
- Verify implementation guides are properly archived after closeout
- Ensure CLAUDE.md accurately reflects current project state
- Verify skills are current and align with actual project patterns

### Phase 4: Vision Alignment

Compare the current state to the project's long-term vision:

- Assess phase progress using the tau:project-management skill (item counts per phase)
- Identify gaps between planned and actual state
- Evaluate whether remaining phases and issues adequately capture the path forward
- Surface risks to long-term objectives

### Phase 5: Infrastructure Adjustments

Address identified discrepancies:

- Create new issues for identified gaps or technical debt
- Close issues that are no longer relevant
- Update phase assignments as needed
- Create or close milestones
- Bootstrap labels on repos that are missing them
- Archive completed context documents
- Update CLAUDE.md or skills if patterns have evolved

### Phase 6: Review Report

Ensure the directory exists:

```bash
mkdir -p .claude/context/reviews
```

Create a review report at `.claude/context/reviews/[YYYY-MM-DD]-[scope].md`:

```markdown
# Project Review - [Scope]

**Date:** [YYYY-MM-DD]
**Scope:** [Repository or project name]

## Infrastructure Audit

| Area | Status | Notes |
|------|--------|-------|
| Project board | [OK / Needs attention] | [Details] |
| Phases | [OK / Needs attention] | [Details] |
| Labels | [OK / Needs attention] | [Details] |
| Milestones | [OK / Needs attention] | [Details] |
| Orphaned issues | [Count] | [Details] |

## Codebase Assessment

| Area | Grade | Notes |
|------|-------|-------|
| Code quality | [A-F] | [Details] |
| Test coverage | [A-F] | [Percentage, critical path coverage] |
| Documentation | [A-F] | [Details] |
| Dependencies | [A-F] | [Details] |
| Technical debt | [A-F] | [Details] |

## Context Health

| Area | Status | Notes |
|------|--------|-------|
| Concepts | [Current / Stale items] | [Details] |
| Guides | [Properly archived / Issues] | [Details] |
| Sessions | [Complete / Missing summaries] | [Details] |
| CLAUDE.md | [Current / Needs update] | [Details] |
| Skills | [Current / Needs update] | [Details] |

## Vision Alignment

| Objective | Status | Notes |
|-----------|--------|-------|
| [Objective from roadmap] | [On track / At risk / Off track] | [Details] |

## Phase Progress

| Phase | Total | Open | Closed | Progress |
|-------|-------|------|--------|----------|
| [Phase name] | [N] | [N] | [N] | [N%] |

## Actions Taken

- [What was created, modified, or archived during this review]

## Recommendations

- [Suggestions for the next review cycle or upcoming work]
```

## Outcomes

At the end of a project review session:

- Review report exists in `.claude/context/reviews/`
- Project-management infrastructure is current and accurate
- Stale context documents are archived
- Missing issues are created for identified gaps
- CLAUDE.md and skills reflect current project state
