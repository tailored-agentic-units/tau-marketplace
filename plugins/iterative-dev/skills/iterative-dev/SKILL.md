---
name: iterative-dev
argument-hint: "[bootstrap | init <issue-number> | review]"
description: >
  Lightweight, plan-mode-driven iterative development workflow with issue-driven sessions.
  Use when the user asks to "bootstrap a project", "start a session", "init session",
  "start development", "review project", "project review", "session closeout",
  "wrap up", or any iterative development activity.
  Triggers: bootstrap, init, session, iterative, review project, project review,
  wrap up, closeout, start development, new project.

  When this skill is invoked, identify the sub-command from the argument
  (bootstrap, init, or review) and follow the corresponding workflow in the
  commands/ directory. Load the github-cli:github-cli skill for issue and PR operations.
---

# Iterative Development Workflow

A lightweight, plan-mode-driven development workflow with issue-driven sessions. Each session maps to one GitHub issue, one branch, and one PR.

## Sub-Commands

Route based on `$ARGUMENTS`:

| Command | File | Purpose |
|---------|------|---------|
| `bootstrap` | [commands/bootstrap.md](commands/bootstrap.md) | Scaffold a new project with the iterative-dev workflow |
| `init <issue-number>` | [commands/init.md](commands/init.md) | Start a development session from a GitHub issue |
| `review` | [commands/review.md](commands/review.md) | Analyze project state and alignment |

Read the corresponding command file and follow its instructions. If no sub-command is provided, explain the available commands and the workflow conventions below.

## Conventions

### `.claude/project/` — The Authoritative Document

The source of truth for the project, organized as a directory of focused files. `README.md` is the index — vision, high-level architecture, and pointers to detail files. Sub-files cover specific concerns (schemas, requirements, configuration, etc.).

Every session reads `README.md` to orient, then loads sub-files as needed. The structure adapts to the project, but generally:

- **README.md** — vision, how it works, high-level architecture, directory of sub-files
- **requirements.md** — checkbox list grouped by area, ordered by dependency
- Additional files as the project's complexity warrants

These documents are alive. Check off completed requirements. Add new ones as they emerge. Keep each file focused on a single concern.

### Issue-Driven Sessions

Each development session is tied to a GitHub issue. The issue body contains the session's scope, approach, constraints, and acceptance criteria. Session continuity flows through the issue chain: each closeout creates the next issue.

- **Bootstrap** creates the first issue
- **Init** reads an issue to start a session
- **Closeout** creates the next issue and closes the current one via PR

### Role Boundaries

**Developer owns:** production source code, configuration files, infrastructure definitions. Source code is written *without* doc comments — those are added by the AI during closeout.

**AI owns:**
- **Tests** — creation and maintenance under `tests/` (or the project's equivalent)
- **Documentation** — repo root `README.md`, project docs (`.claude/project/` or equivalent like `_project/`), `CLAUDE.md`, memory files
- **Source comments** — godoc/jsdoc/docstrings on exported types, functions, methods; explanatory inline comments where logic is non-obvious
- **Project-management artifacts** — commit messages, PR bodies, issue creation/edits, branch labels
- **Implementation guides** — the reference document the developer follows

The boundary is concrete: if a line lives in a `.go`/`.ts`/`.py` source file, the developer writes it (without comments); if it lives anywhere else — docs, comments, tests, prose — the AI writes it.

During init sessions, the AI creates an implementation guide after plan mode. The developer executes the guide. The AI resumes for testing, validation, documentation, and closeout.

### Implementation Guides

Created during init sessions at `.claude/context/guides/[issue-number]-[slug].md`. Contains problem context, architecture approach, and step-by-step implementation. Archived to `.claude/context/guides/.archive/` at session closeout.

Guide conventions:

- **Existing files**: show incremental changes only (never replace entire files)
- **New files**: provide the structural skeleton — types, function signatures, function bodies — *without* doc comments. Code blocks should read as design intent, not as final source.
- **Exclude entirely from guide code blocks:**
  - Godoc / jsdoc / docstrings — the AI adds these during closeout
  - Test code — the AI writes tests during closeout
  - Documentation updates of any kind — root `README.md`, project docs, `CLAUDE.md`, memory files all stay out of the guide
  - Project-management artifacts — issue rewrites, label changes, commit messages, PR bodies
- **Guide-level commentary belongs in the guide's prose** between code blocks. Explain how a piece fits the larger picture, reference other steps, flag integration concerns. This commentary lives in the guide only — it MUST NOT be transcribed into source files as comments.

**Litmus test:** if a line would still belong in the file six months later via `git blame`, it's source code (the developer writes it during execution, the AI adds doc comments during closeout). If it's only useful *while reading the guide*, it stays in the guide as prose (the AI writes it once when authoring the guide).

### Branching

Each development session works on its own branch, created from `main` at session start. Branch names follow the `[issue-number]-[slug]` pattern (e.g., `7-state-architecture`, `12-codex-layer`).

### The Development Loop

```
Start session (init <issue-number>)
 └─ Read issue for session context
    └─ Create branch [issue-number]-[slug] from main
       └─ Read .claude/project/ for context
          └─ Plan in plan mode
             └─ Create implementation guide
                └─ STOP — developer executes guide
                   └─ AI resumes: testing, validation, docs
                      └─ Pre-commit review
                         ├─ Reconcile project docs
                         ├─ Discuss next steps, create next issue
                         └─ Commit, push, PR (closes current issue)
```

### Pre-Commit Review

Before committing, run a structured closeout:

1. **Reconcile project docs** — review `.claude/project/` against the session's changes. Check off completed requirements, update architecture/schema/state docs, add new sub-files if new concepts emerged. The goal: someone reading the project docs after this session sees a coherent, current picture.

2. **Discuss next steps** — have a genuine conversation with the user about what comes next:
   - If the next step is a **development session**: identify the next build target. Surface new ideas or directions beyond current requirements.
   - If there's **no obvious next step**: discuss openly. Review project state, what's working, what's incomplete. The conversation itself often reveals the next direction.

3. **Create next issue** — capture the discussion's outcome as a new GitHub issue with scope, approach, and acceptance criteria. Tell the user the issue number.

4. **Commit, push, and PR** — open a pull request that closes the current issue.
