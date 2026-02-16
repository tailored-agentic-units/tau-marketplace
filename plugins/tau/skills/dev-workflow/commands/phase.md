# Phase Planning Session

## Purpose

Decompose a Phase into Objectives. Phase planning takes a phase-level focus area
(produced by concept development) and creates structured Objective issues that
drive objective planning and task execution sessions.

| Input | Output |
|-------|--------|
| A Phase (from concept development) | Objective issues on the primary repo, `_project/phase.md` |

## Prerequisites

- Load skills: **tau:project-management**, **tau:github-cli**
- Load domain-specific skills for architecture context as needed

## Workflow Lifecycle

```
Concept Development → Phase Planning → Objective Planning → Task Execution
(produces Phases)     ^^^^^^^^^^^^^^
                      This session type
```

## Invocation

```
/dev-workflow phase
```

## Session Metadata

1. **Phase name** — identified from the Phases roadmap in `_project/README.md` or collected from the user
2. **Version target** — the semantic version this phase builds toward (from the Phases roadmap, e.g., `v0.1.0`)
3. **Primary repository** — where Objectives will be created
4. **`_project/README.md`** — read from the primary repository for vision, architecture, and the Phases roadmap

---

## Workflow

### Step 0: Transition Closeout

> Only runs when `_project/phase.md` already exists, indicating a previous phase
> is being transitioned out.

#### 0a. Status Assessment

- Read `_project/phase.md` to identify the current phase name, version target, and objectives
- Read `_project/README.md` to consult the Phases roadmap — identify where the completed phase sits and which phase comes next
- Query the project board for objective completion status in the current phase
- Present a status summary to the user:
  - Phase name and version target
  - Objectives: completed vs. open
  - For each open objective: sub-issue completion status
  - Next phase from the roadmap (suggested based on the Phases table in `_project/README.md`)

#### 0b. Disposition of Incomplete Objectives

> Only if open objectives exist.

For each open objective:

- **All sub-issues complete** — close the objective issue:

  ```bash
  gh issue close <objective-number> --repo <owner>/<primary-repo>
  ```

- **Open sub-issues remain** — ask the user for each open objective: **transition to next phase** or **move to backlog**?

  - **Backlog**: update phase assignment to "Backlog" on the project board for the objective and all its sub-issues:

    ```bash
    # Resolve IDs
    BACKLOG_OPTION_ID=$(gh project field-list <project-number> --owner <owner> \
      --format json --jq '.fields[] | select(.name == "Phase") | .options[] | select(.name == "Backlog") | .id')

    # Update objective item
    gh project item-edit --id <item-id> --project-id "$PROJECT_ID" \
      --field-id "$FIELD_ID" --single-select-option-id "$BACKLOG_OPTION_ID"
    # Repeat for each sub-issue item
    ```

  - **Transition**: note the objective for phase reassignment after the new phase is created (Step 6)

#### 0c. Clean Slate

- Delete `_project/phase.md` — GitHub project infrastructure serves as the historical record
- Delete `_project/objective.md` if it exists (belongs to the outgoing phase)

---

### 1. Context Gathering

- Read `_project/README.md` from the primary repository for vision, architecture, and the Phases roadmap to identify this phase's area of focus and version target
- Review existing concept documents in `.claude/context/concepts/`
- Review the project board: current phase items, backlog, existing objectives
- Identify relevant domain skills and load them for architecture context

### 2. Phase Decomposition

- Identify the cohesive sets of requirements within the phase
- Each set becomes an Objective — a parent issue that aggregates related work
- Consider cross-repo boundaries: which repositories are involved in each Objective
- Order Objectives by dependency (earlier Objectives may unblock later ones)

### 3. Collaborative Refinement

- Present proposed Objectives to the user
- Discuss scope boundaries, ordering, and dependencies
- Iterate until the decomposition is aligned

### 4. Objective Creation

Create each Objective as a parent issue on the primary repository. Objectives receive
the `objective` label (no category or package labels) and the `Objective` issue type.

**Objective issue body convention:**

Each Objective body should contain enough context to drive an Objective Planning session:

```bash
OBJECTIVE_URL=$(gh issue create \
  --repo <owner>/<primary-repo> \
  --title "Objective: <title>" \
  --label objective \
  --milestone "<phase-name>" \
  --body "$(cat <<'EOF'
## Objective

[What this objective achieves and why]

## Scope

[What is in scope, what is not]

## Repositories

[Which repos are involved and what each contributes]

## Dependencies

[Other objectives or prerequisites this depends on]
EOF
)" --json url --jq '.url')

# Assign the Objective issue type
ISSUE_ID=$(gh issue view "$OBJECTIVE_URL" --json id --jq '.id')
gh api graphql -f query='
mutation($issueId: ID!, $typeId: ID!) {
  updateIssueIssueType(input: { issueId: $issueId, issueTypeId: $typeId }) {
    issue { number issueType { name } }
  }
}' -f issueId="$ISSUE_ID" -f typeId="<objective-type-id>"
```

### 5. Update `_project/phase.md`

Create `_project/phase.md` with:

- Phase name and version target
- Phase scope and goals
- Objectives table (with status) — this table serves as the objective roadmap for subsequent objective planning sessions
- Constraints and cross-cutting decisions

### 6. Project Board Updates

- Add each Objective to the project board
- Assign to the appropriate phase
- Verify milestones are set on the primary repository
- **If transitioning**: update phase assignment for carried-forward objectives from Step 0b to the new phase

  ```bash
  NEW_OPTION_ID=$(gh project field-list <project-number> --owner <owner> \
    --format json --jq '.fields[] | select(.name == "Phase") | .options[] | select(.name == "<new-phase>") | .id')

  gh project item-edit --id <item-id> --project-id "$PROJECT_ID" \
    --field-id "$FIELD_ID" --single-select-option-id "$NEW_OPTION_ID"
  # Repeat for each carried-forward objective and its sub-issues
  ```

- Close any objectives from the old phase that are now fully resolved (all sub-issues complete, no remaining carry-forward work)

## Outcomes

- Version target established for the phase (used for dev pre-release tags)
- Objectives created on the primary repository with the `objective` label and `Objective` issue type
- Objectives added to the project board and assigned to the phase
- Each Objective has enough context to drive an Objective Planning session
- `_project/phase.md` created with phase scope, objectives table (serving as objective roadmap), and version target
- Previous phase cleaned up (if transitioning)
- Incomplete work from the previous phase explicitly handled (transitioned or moved to backlog)
