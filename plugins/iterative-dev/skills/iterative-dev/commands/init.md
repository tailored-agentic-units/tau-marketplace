# Start a Development Session

Begin a new session from a GitHub issue. The issue number is provided in `$ARGUMENTS`.

## Stage 1: Session Setup

### Read the Issue

Parse the issue number from `$ARGUMENTS` and read the issue for session context:

```bash
gh issue view <number> --json title,body,labels
```

The issue body contains the session's scope, approach, constraints, and acceptance criteria.

### Create a Session Branch

```bash
git checkout main
git pull
git checkout -b <issue-number>-<descriptive-slug>
```

Branch naming: `[issue-number]-[title-slug]` in kebab-case. Example: `7-state-architecture`

### Read Project Context

Read `.claude/project/README.md` for project orientation:
- Vision and high-level architecture
- Pointers to detail files for deeper context

Load relevant sub-files from `.claude/project/` based on what the session touches (e.g., `requirements.md` for the checklist, `schemas.md` if working on state, etc.).

## Stage 2: Plan and Guide

### Plan the Implementation

Enter plan mode. Collaborate with the user on:
- How to approach the work defined in the issue
- Implementation details — which files to create/modify, what order
- Design decisions that need to be made
- Anything in the issue that needs refinement based on current project state

The user trusts the AI with implementation details after alignment in the planning phase. Focus planning on conceptual alignment, not line-by-line code review.

### Create Implementation Guide

**CRITICAL: Do NOT begin implementing code changes.** When the plan is approved, the next action is to write an implementation guide document. This guide is a reference artifact for the developer to follow. Do not edit source files, do not begin coding. Write the guide document and then stop.

Ensure the directory exists:

```bash
mkdir -p .claude/context/guides
```

Write to `.claude/context/guides/[issue-number]-[slug].md`:

```markdown
# [Issue Number] - [Title]

## Problem Context

[From the issue body - why this work is needed]

## Architecture Approach

[From the plan discussion - what was decided and why]

## Implementation

### Step 1: [Description]

[Complete implementation for this step. If the step affects multiple source files,
show the implementation for each file. Provide actual code changes, not just
descriptions of what should be updated.]

### Step 2: [Description]

...

## Remediation

[Added during developer execution if blockers are discovered.
Each remedial step is numbered R1, R2, etc.]

### R1: [Description of blocker and fix]

...

## Validation Criteria

- [ ] [Specific checks that confirm correct implementation]
```

**File change conventions:**

- **Existing files**: Show incremental changes only (what is being added or modified). Never replace entire existing files.
- **New files**: Provide the structural skeleton — types, function signatures, function bodies — *without* doc comments. Code blocks should read as design intent, not as final source.
- **Exclude entirely from guide code blocks:**
  - Godoc / jsdoc / docstrings — the AI adds these during closeout
  - Test code — the AI writes tests during closeout
  - Documentation updates of any kind — root `README.md`, project docs, `CLAUDE.md`, memory files all stay out of the guide
  - Project-management artifacts — issue rewrites, label changes, commit messages, PR bodies
- **Guide-level commentary belongs in the guide's prose** between code blocks. Explain how a piece fits the larger picture, reference other steps, flag integration concerns. This commentary lives in the guide only — it MUST NOT be transcribed into source files as comments.

**Litmus test:** if a line would still belong in the file six months later via `git blame`, it's source code (the developer writes it during execution, the AI adds doc comments during closeout). If it's only useful *while reading the guide*, it stays in the guide as prose (the AI writes it once when authoring the guide).

**After writing the guide, stop and wait.** The developer executes the guide independently. The AI remains available for mentorship, clarification, and adjustments. If blockers are discovered, add a **Remediation** section to the guide (numbered R1, R2, etc.).

## Stage 3: AI Closeout

Control returns to the AI when the developer signals that implementation is complete.

### Testing

- Add tests for new functionality
- Update existing tests affected by the implementation
- Run the test suite and ensure all tests pass

### Validation

- Review implementation against the guide
- Verify acceptance criteria from the issue are met
- Run language-specific quality checks (lint, vet, etc.)

### Documentation

This is where godoc/jsdoc/docstring comments enter the source files. The implementation guide deliberately omits them so guide code blocks read as design intent; the AI adds them now to the actual source.

- Add doc comments to exported types, functions, and methods in every file the session created or substantively modified
- Document non-obvious behavior, invariants, or design decisions in inline comments where they aid future readers
- **Do not** add comments that describe decision history ("X chosen over Y because...", "considered Z and rejected"). Decision rationale belongs in commit messages, PR bodies, or design docs — never in source comments

### Pre-Commit Review

#### Reconcile Documentation

Review the session's full doc surface area for updates:

- **Project docs** — `.claude/project/` or the project's equivalent (e.g., `_project/`). Check off completed items in `requirements.md`. Update architecture/schema/configuration docs if structure changed. Add new focused sub-files if new concepts emerged.
- **Repo root `README.md`** — does the project structure tree, the API surface, the feature list, or the getting-started instructions need updating?
- **`CLAUDE.md`** — did this session establish new conventions, patterns, principles, or architectural rules that should be captured for future sessions?
- **Memory files** — did the user surface feedback or a principle that should persist across sessions and projects?

Goal: a reader picking up the project after this session sees a coherent, current picture across all doc surfaces — not just the project directory.

#### Discuss Next Steps

Have a conversation with the user about what comes next:

**If the next step is a development session:**
- What's the next logical build target from remaining requirements?
- Are there new ideas or directions beyond current requirements?

**If there's no obvious next step:**
- Discuss openly. Review the project's current state, what's working well, what feels incomplete. The conversation often reveals the next direction.

#### Create Next Issue

From the discussion, create the next GitHub issue:

```bash
gh issue create --title "<title>" --body "<scope, approach, acceptance criteria>"
```

Tell the user the new issue number for their next session.

#### Archive Implementation Guide

```bash
mkdir -p .claude/context/guides/.archive
mv .claude/context/guides/<issue-number>-<slug>.md .claude/context/guides/.archive/
```

### Commit, Push, and PR

```bash
git add .
git commit -m "<message>"
git push -u origin <issue-number>-<slug>
gh pr create --title "<concise summary>" --body "$(cat <<'EOF'
## Summary

[What was implemented and why]

Closes #<issue-number>
EOF
)"
```
