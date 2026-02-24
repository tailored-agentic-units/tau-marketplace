# Objective Planning Session

## Purpose

Decompose an Objective into executable sub-issues. Objective planning takes a
parent Objective issue and produces sub-issues across the relevant repositories,
each representing a single development session (one branch, one PR).

| Input | Output |
|-------|--------|
| An Objective issue | Sub-issues across relevant repos, `_project/objective.md` |

## Prerequisites

- Load skills: **tau:project-management**, **tau:github-cli**
- Load domain-specific skills for architecture context as needed

## Workflow Lifecycle

```
Concept Development → Phase Planning → Objective Planning → Task Execution
(produces Phases)                      ^^^^^^^^^^^^^^^^^^^
                                       This session type
```

## Invocation

```
/dev-workflow objective <issue-number>
```

## Session Metadata

1. **Issue number** — the Objective (parent issue) to plan
2. **Target repositories** — identified from the Objective body or collected

---

## Workflow

### Step 0: Transition Closeout

> Only runs when `_project/objective.md` already exists, indicating a previous
> objective is being transitioned out.

#### 0a. Status Assessment

- Read `_project/objective.md` to identify the current objective (issue number, repo, sub-issues)
- Read `_project/phase.md` to consult the objectives table — understand where the completed objective sits and which objective comes next in the phase roadmap
- Query GitHub for sub-issue completion status:

  ```bash
  PARENT_ID=$(gh issue view <previous-objective-number> --repo <owner>/<primary-repo> --json id --jq '.id')
  gh api graphql \
    -H "GraphQL-Features: sub_issues" \
    -f query='query($id: ID!) {
      node(id: $id) {
        ... on Issue {
          number title state
          subIssues(first: 50) {
            nodes { number title state url repository { nameWithOwner } }
          }
          subIssuesSummary { total completed percentCompleted }
        }
      }
    }' -f id="$PARENT_ID"
  ```

- Present status summary to user:
  - Objective title and issue number
  - Sub-issue completion: N/M complete (X%)
  - List of open sub-issues with their titles and repos

#### 0b. Disposition of Incomplete Work

> Only if open sub-issues exist.

- Warn user about open sub-issues
- Ask user to choose for each open sub-issue: **carry forward** to the new Objective, or **move to backlog**?

  - **Backlog**: remove from parent objective and update phase to "Backlog" on the project board:

    ```bash
    # Remove sub-issue from parent
    gh api graphql \
      -H "GraphQL-Features: sub_issues" \
      -f query='mutation($parentId: ID!, $childId: ID!) {
        removeSubIssue(input: {issueId: $parentId, subIssueId: $childId}) {
          subIssue { number title }
        }
      }' -f parentId="$PARENT_ID" -f childId="$CHILD_ID"

    # Update phase to Backlog on project board
    gh project item-edit --id <item-id> --project-id "$PROJECT_ID" \
      --field-id "$FIELD_ID" --single-select-option-id "$BACKLOG_OPTION_ID"
    ```

  - **Carry forward**: note the sub-issue IDs — they will be re-parented in Step 4 after the new Objective is created

#### 0c. Prepare Clean Slate

- Update `_project/phase.md` objectives table — mark previous objective's status
- Delete `_project/objective.md` — GitHub project infrastructure serves as the historical record

---

### 1. Context Gathering

- Read the Objective issue for scope, architecture context, and dependencies
- Read `_project/README.md` and `_project/phase.md` for project and phase context
- Review relevant concept documents in `.claude/context/concepts/`
- Explore the affected repositories to understand current state
- Load domain-specific skills for architecture decisions
- Note any carry-forward sub-issues from Step 0b that will be incorporated

### 2. Architecture Discussion

- Identify the specific changes needed across each repository
- Discuss architectural approaches and trade-offs
- Define interfaces and contracts between components
- Resolve open design questions relevant to this Objective
- Consider how carry-forward sub-issues (if any) fit into the new Objective's architecture

### 3. Sub-Issue Decomposition

- Break the Objective into individual development tasks
- Each sub-issue should be resolvable in a single development session
- Each sub-issue maps to exactly **one branch and one PR** on its repository
- Order sub-issues by dependency where possible
- Incorporate carry-forward sub-issues into the ordering (they may already have partial context)

### 4. Sub-Issue Creation

Create sub-issues on their respective repositories and link to the Objective. Sub-issues
receive category label(s) and package label(s) (not the `objective` label) and the
`Task` issue type (or `Bug` for defect fixes).

**Sub-issue body convention:**

Each sub-issue body should contain enough context to bootstrap a task execution session:

```bash
# Create sub-issue on the target repo
CHILD_URL=$(gh issue create \
  --repo <owner>/<target-repo> \
  --title "<sub-issue title>" \
  --label "<category>" --label "<package>" \
  --milestone "<phase-name>" \
  --body "$(cat <<'EOF'
## Context

[How this sub-issue fits into the parent Objective]

## Scope

[What this sub-issue covers]

## Approach

[Recommended implementation strategy]

## Acceptance Criteria

- [ ] [Specific, verifiable criteria]
EOF
)")

# Assign the Task issue type (or Bug for defect fixes)
CHILD_ID=$(gh issue view "$CHILD_URL" --json id --jq '.id')
gh api graphql -f query='
mutation($issueId: ID!, $typeId: ID!) {
  updateIssueIssueType(input: { issueId: $issueId, issueTypeId: $typeId }) {
    issue { number issueType { name } }
  }
}' -f issueId="$CHILD_ID" -f typeId="<task-type-id>"

# Link to the Objective
PARENT_ID=$(gh issue view <objective-number> --repo <owner>/<primary-repo> --json id --jq '.id')
gh api graphql \
  -H "GraphQL-Features: sub_issues" \
  -f query='mutation($parentId: ID!, $childId: ID!) {
    addSubIssue(input: {issueId: $parentId, subIssueId: $childId}) {
      subIssue { number title url }
    }
  }' -f parentId="$PARENT_ID" -f childId="$CHILD_ID"
```

**Re-parent carry-forward sub-issues** (if any from Step 0b):

```bash
# Atomically move sub-issue from old objective to new objective
gh api graphql \
  -H "GraphQL-Features: sub_issues" \
  -f query='mutation($parentId: ID!, $childId: ID!) {
    addSubIssue(input: {issueId: $parentId, subIssueId: $childId, replaceParent: true}) {
      subIssue { number title url }
    }
  }' -f parentId="$NEW_OBJECTIVE_ID" -f childId="$CARRY_FORWARD_ID"
```

**Close previous Objective** (after all carry-forward sub-issues are re-parented):

```bash
# Now safe — all sub-issues are either complete, re-parented, or in backlog
gh issue close <previous-objective-number> --repo <owner>/<primary-repo>
```

### 5. Update `_project/objective.md`

Create `_project/objective.md` with:

- Objective title and parent issue reference
- Phase reference
- Scope and acceptance criteria
- Sub-issues table (with status, noting any that were carried forward)
- Architecture decisions specific to this objective

### 6. Project Board Updates

- Add each sub-issue to the project board
- Assign to the same phase as the parent Objective
- Verify milestones are set on each repository

## Outcomes

- Sub-issues created on their respective repositories with bootstrap context
- Sub-issues assigned the `Task` (or `Bug`) issue type and linked to the Objective via GraphQL API
- Each sub-issue has enough context to drive a Task Execution session
- Architecture decisions documented in the sub-issue bodies
- `_project/objective.md` created with scope, sub-issues, and architecture decisions
- Previous objective closed and its incomplete work explicitly handled (re-parented or moved to backlog)
