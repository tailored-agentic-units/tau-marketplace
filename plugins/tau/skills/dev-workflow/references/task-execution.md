# Task Execution Session

## Purpose

Execute a development task driven by a GitHub issue, from planning through implementation to structured closeout. This workflow integrates Claude Code plan mode, implementation guides, and project-management updates.

## Prerequisites

- Load skills: **tau:github-cli**, plus domain-specific skills based on development type
- Collect session metadata: issue number (required), development type (auto-detect or collect)
- Identify and load the appropriate dev-type reference from `references/dev-types/`

## Development Type References

Dev-type references augment this workflow with type-specific conventions for implementation, testing, and validation. Load the appropriate reference based on the development type identified during session initialization.

Available dev-type references:

- For Go library development, see [go-library.md](dev-types/go-library.md)

Additional dev-type references are added incrementally as needed in the `dev-types/` directory.

**How dev-type references integrate with this workflow:**

| Phase | Dev-Type Augmentation |
|-------|----------------------|
| Phase 2: Plan Mode | Load domain skills specified by the dev-type reference |
| Phase 3: Guide Creation | Apply dev-type file conventions and code patterns |
| Phase 5: Validation | Execute dev-type testing strategy and validation checklist |
| Phase 6: Documentation | Follow dev-type documentation conventions |

## Workflow

### Phase 1: Session Initialization

1. Read the issue to bootstrap context:

```bash
gh issue view <number> --json title,body,labels,milestone,assignees
```

2. Parse the issue body for context (scope, approach, dependencies, acceptance criteria)

3. Create a branch from the issue:

```bash
git checkout -b <issue-number>-<title-slug>
```

Branch naming: `[issue-number]-[title-slug]` in kebab-case. Example: `42-audio-protocol`

4. Identify the development type from issue labels and load the corresponding dev-type reference

5. Load domain skills specified by the dev-type reference (e.g., tau:go-patterns for Go development)

### Phase 2: Plan Mode Exploration

1. Enter Claude Code plan mode
2. Explore the codebase to understand current state and affected files
3. Identify integration points and dependencies
4. Discuss implementation strategy collaboratively, applying patterns from loaded domain skills
5. Iterate until approach is aligned - this is not a one-shot plan presentation

### Phase 3: Implementation Guide Creation

**CRITICAL: Do NOT begin implementing code changes.** When the plan is approved, the next action is to write an implementation guide document. This guide is a reference artifact for the developer to follow during Phase 4. Do not create tasks, do not edit source files, do not begin coding. Write the guide document and then stop and wait for the developer to execute it.

Ensure the directory exists before writing:

```bash
mkdir -p .claude/context/guides
```

**Location:** `.claude/context/guides/[issue-number]-[slug].md`

**Guide structure:**

```markdown
# [Issue Number] - [Title]

## Problem Context

[From the issue body - why this work is needed]

## Architecture Approach

[From the plan discussion - what was decided and why]

## Implementation

### Step 1: [Description]

[Step-by-step implementation instructions ordered by dependency level]

### Step 2: [Description]

...

## Validation Criteria

- [ ] [Specific checks that confirm correct implementation]
```

**File change conventions:**

- **Existing files**: Show incremental changes only (what is being added or modified). Never replace entire existing files.
- **New files**: Provide complete implementation.

**Code block conventions:**

- **No godoc comments.** Godoc comments are added by the AI during Phase 6 (Documentation). Omitting them from the guide saves tokens and avoids maintenance.
- **No unit tests.** Tests are implemented by the AI during Phase 5 (Validation). Do not include test code in the guide.
- **Meaningful comments only.** Comments that help the developer understand intent, non-obvious logic, or integration context are encouraged. Boilerplate or self-evident comments should be omitted.

Apply any additional file conventions specified by the dev-type reference.

### Phase 4: Developer Execution

**After writing the implementation guide, stop and wait.** The developer executes the guide independently. Do not proceed to Phase 5 until the developer signals that implementation is complete.

- Developer executes the implementation as the code base maintainer
- AI remains available for mentorship, clarification, and adjustments
- Focus on code structure and correctness

### Phase 5: Validation

Control transitions back to the AI for the remainder of the session.

**Core validation (all development types):**

- Review implementation for accuracy and completeness against the guide
- Verify acceptance criteria from the issue are met

**Dev-type validation (from the loaded dev-type reference):**

- Execute the dev-type testing strategy (e.g., add/revise tests, run test suites)
- Run through the dev-type validation checklist
- Ensure all dev-type quality gates pass before proceeding

The dev-type reference defines the specific testing infrastructure, commands, and coverage expectations for this phase. The AI is responsible for implementing any testing changes required as a result of the implementation and ensuring all tests pass.

### Phase 6: Documentation

- Add documentation comments to exported types, functions, and methods
- Document non-obvious behavior or design decisions
- Follow documentation conventions from the dev-type reference
- Update relevant documentation files if the changes affect them

### Phase 7: Closeout

#### 7a. Create Session Summary

Ensure the directory exists:

```bash
mkdir -p .claude/context/sessions
```

Create a session summary at `.claude/context/sessions/[issue-number]-[slug].md`:

```markdown
# [Issue Number] - [Title]

## Summary

[Brief description of what was implemented]

## Key Decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| [Decision point] | [What was chosen] | [Why] |

## Files Modified

- [List of files created or modified]

## Patterns Established

- [Any new patterns introduced that future sessions should follow]

## Validation Results

- [Test results, coverage metrics, manual verification]
```

#### 7b. Archive Implementation Guide

```bash
mkdir -p .claude/context/guides/.archive
mv .claude/context/guides/[issue-number]-[slug].md .claude/context/guides/.archive/
```

#### 7c. Pull Request

Commit all changes, push the branch to the remote, and create a PR:

```bash
git add .
git commit -m "<message>"
git push -u origin <issue-number>-<title-slug>
```

Then create the PR using the tau:github-cli skill:

```bash
gh pr create --title "[concise summary]" --body "$(cat <<'EOF'
## Summary

[What was implemented and why]

Closes #[issue-number]

## Changes

- [Key changes]
EOF
)"
```

#### 7d. Project-Management Update

- Update phase assignment if the work completes a phase milestone
- Close the issue if all acceptance criteria are met (the PR close will handle this if `Closes #N` is in the body)
