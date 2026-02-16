# Plan: Split Planning into Phase & Objective Commands + Transition Logic

## Context

The dev-workflow skill currently routes phase and objective planning through a single `planning.md` command with a `plan` prefix (`/dev-workflow plan phase`, `/dev-workflow plan objective`). After completing an Objective, there's no mechanism to close it out and transition to the next one — no issue closure, no cleanup of `_project/objective.md`, no handling of incomplete sub-issues. This change:

1. Splits planning into dedicated `phase` and `objective` sub-commands with transition closeout logic
2. Renames all command files to match their command names directly (e.g., `concept-development.md` → `concept.md`)
3. Follows the tau-marketplace-dev update workflow (branch, version bump 0.0.8 → 0.1.0, CHANGELOG, PR)

## Marketplace Update Workflow

Per `tau-marketplace-dev`, the update session follows:
1. Create branch: `update/split-planning-phase-objective-commands`
2. Execute all file changes below
3. Bump version in `plugins/tau/.claude-plugin/plugin.json` → `0.1.0`
4. Add `## 0.1.0` entry to `CHANGELOG.md`
5. Commit, push, create PR

## File Changes

### Renames (command files → match command names)

| Current | New |
|---------|-----|
| `commands/concept-development.md` | `commands/concept.md` |
| `commands/project-review.md` | `commands/review.md` |
| `commands/task-execution.md` | `commands/task.md` *(includes existing unstaged remediation changes)* |
| `commands/release.md` | `commands/release.md` *(no change — already matches)* |

### Delete

| File | Reason |
|------|--------|
| `commands/planning.md` | Replaced by `phase.md` and `objective.md` |

### Create

| File | Source |
|------|--------|
| `commands/phase.md` | Phase Planning section from `planning.md` + transition closeout |
| `commands/objective.md` | Objective Planning section from `planning.md` + transition closeout |

### Modify

| File | Changes |
|------|---------|
| `SKILL.md` | Argument hint, routing table, triggers, session descriptions, file references |
| `plugins/tau/.claude-plugin/plugin.json` | Version `0.0.8` → `0.1.0` |
| `CHANGELOG.md` | Add `## 0.1.0` entry |

---

## SKILL.md Changes

**Argument hint:**
```
[concept | phase | objective <issue-number> | task <issue-number> | review | release <version>]
```

**Routing table:**

| Pattern | Session Type |
|---------|-------------|
| `/dev-workflow concept` | Concept Development |
| `/dev-workflow phase` | Phase Planning |
| `/dev-workflow objective <issue>` | Objective Planning |
| `/dev-workflow task <issue-number>` | Task Execution |
| `/dev-workflow review` | Project Review |
| `/dev-workflow release <version>` | Release |

**Session type descriptions** — replace the unified "Planning Session" section with separate "Phase Planning Session" and "Objective Planning Session" sections, each referencing their new command file.

**Trigger keywords** — replace `plan, planning, plan phase, plan objective` with `phase, phase planning, objective, objective planning`.

**Skill Integration table** — split "Planning" row into "Phase Planning" and "Objective Planning" (both load tau:project-management, tau:github-cli).

**All file references** updated: `planning.md` → removed, `concept-development.md` → `concept.md`, `project-review.md` → `review.md`, `task-execution.md` → `task.md`.

---

## Transition Closeout Design

### Principle: No Orphaned Issues

When closing an objective or phase, incomplete work must be explicitly handled — either transitioned to the next objective/phase or moved to backlog. Issues are only closed when truly complete.

### Objective Transition (in `commands/objective.md`)

Prepended as **Step 0** before the standard objective planning workflow. Only runs when `_project/objective.md` exists.

**0a. Status Assessment**
- Read `_project/objective.md` to identify current objective (issue number, repo)
- Query GitHub for sub-issue completion status via GraphQL (`subIssues` query)
- Present status summary to user

**0b. Disposition of Incomplete Work** *(only if open sub-issues exist)*
- Warn user about open sub-issues with their titles and status
- Ask user to choose per sub-issue: **carry forward** to the new Objective, or **move to backlog**
- Backlog items: remove from parent objective (`removeSubIssue`), update phase assignment to "Backlog" on project board
- Carry-forward items: note their issue IDs (will be re-parented in Step 4 after new Objective is created)

**0c. Prepare Clean Slate**
- Update `_project/phase.md` objectives table — mark previous objective's status
- Clear `_project/objective.md`

**Step 4 additions** (after creating the new Objective issue):
- Re-parent carry-forward sub-issues to the new Objective using `addSubIssue` with `replaceParent: true`
- Close the previous Objective issue (now safe — all sub-issues are either complete, re-parented, or in backlog)

### Phase Transition (in `commands/phase.md`)

Prepended as **Step 0** before the standard phase planning workflow. Only runs when `_project/phase.md` exists.

**0a. Status Assessment**
- Read `_project/phase.md` to identify current phase
- Query project board for objective completion status in current phase
- Present status summary to user

**0b. Disposition of Incomplete Objectives** *(only if open objectives exist)*
- For each open objective:
  - If all sub-issues complete → close the objective issue
  - If open sub-issues exist → ask user: **transition to next phase** or **move to backlog**?
  - Backlog: update phase assignment to "Backlog" on project board for the objective and its sub-issues
  - Transition: note for phase reassignment after new phase is created (Step 6)

**0c. Archive and Clean**
- Archive `_project/phase.md` → `_project/.archive/phase.NN.md`
- Clear `_project/objective.md` if it exists

**Step 6 addition** (after project board updates for new phase):
- Update phase assignment for carried-forward objectives to the new phase
- Close any objectives from the old phase that are now fully resolved

---

## Content Split from `planning.md`

### `commands/phase.md` receives:
- Purpose section (rewritten for phase-specific focus)
- Prerequisites (same: tau:project-management, tau:github-cli)
- Lifecycle context (concept → **phase** → objective → task)
- **NEW: Step 0 — Transition Closeout** (described above)
- Phase Planning session metadata (lines 48-51)
- Phase Planning workflow steps 1-6 (lines 55-133)
- Phase Planning outcomes (lines 135-141)
- Step 4 and Step 6 additions for carry-forward handling

### `commands/objective.md` receives:
- Purpose section (rewritten for objective-specific focus)
- Prerequisites (same: tau:project-management, tau:github-cli)
- Lifecycle context (concept → phase → **objective** → task)
- **NEW: Step 0 — Transition Closeout** (described above)
- Objective Planning session metadata (lines 155-156)
- Objective Planning workflow steps 1-6 (lines 160-252)
- Objective Planning outcomes (lines 254-259)
- Step 4 additions for carry-forward re-parenting and previous objective closure

---

## Verification

1. **File structure**: `ls commands/` shows `concept.md`, `phase.md`, `objective.md`, `task.md`, `review.md`, `release.md` — no old names remain
2. **SKILL.md consistency**: All routing table entries, session descriptions, and file references point to new filenames
3. **No stale references**: `grep -r "planning.md\|concept-development\|project-review\|task-execution" plugins/tau/skills/dev-workflow/` returns nothing
4. **Version**: `plugin.json` shows `0.1.0`, CHANGELOG has `## 0.1.0` entry
5. **Staged changes preserved**: `task.md` includes the remediation convention from the existing unstaged diff
