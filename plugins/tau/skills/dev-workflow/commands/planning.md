# Planning Session

## Purpose

Break down project structure into actionable development units. Planning sessions
translate the phase-level roadmap from concept development into structured GitHub
issues that drive development sessions.

Two variants exist, forming a top-down decomposition:

| Variant | Input | Output |
|---------|-------|--------|
| **Phase Planning** | A Phase (from concept development) | Objective issues on the primary repo, `_project/phase.md` |
| **Objective Planning** | An Objective issue | Sub-issues across relevant repos, `_project/objective.md` |

## Prerequisites

- Load skills: **tau:project-management**, **tau:github-cli**
- Load domain-specific skills for architecture context as needed

## Workflow Lifecycle

Planning sessions fit into the broader development lifecycle:

```
Concept Development → Phase Planning → Objective Planning → Task Execution
(produces Phases)     ^^^^^^^^^^^^^^   ^^^^^^^^^^^^^^^^^^^
                      This session type
```

- **Concept Development** establishes vision, phases, and project infrastructure
- **Phase Planning** decomposes a Phase into Objectives
- **Objective Planning** decomposes an Objective into executable sub-issues
- **Task Execution** resolves a single sub-issue (one branch, one PR)

---

## Phase Planning

### Invocation

```
/dev-workflow plan phase
```

### Session Metadata

1. **Phase name** — identified from the project board or collected from the user
2. **Version target** — the semantic version this phase builds toward (e.g., `v0.1.0`)
3. **Primary repository** — where Objectives will be created
4. **`_project/README.md`** — read from the primary repository for vision and architecture context

### Workflow

#### 1. Context Gathering

- Read `_project/README.md` from the primary repository for vision, architecture, and the Phases section (produced by concept development) to understand this phase's area of focus and version target
- Read `_project/phase.md` if it exists for current phase context
- Review existing concept documents in `.claude/context/concepts/`
- Review the project board: current phase items, backlog, existing objectives
- Identify relevant domain skills and load them for architecture context

#### 2. Phase Decomposition

- Identify the cohesive sets of requirements within the phase
- Each set becomes an Objective — a parent issue that aggregates related work
- Consider cross-repo boundaries: which repositories are involved in each Objective
- Order Objectives by dependency (earlier Objectives may unblock later ones)

#### 3. Collaborative Refinement

- Present proposed Objectives to the user
- Discuss scope boundaries, ordering, and dependencies
- Iterate until the decomposition is aligned

#### 4. Objective Creation

Create each Objective as a parent issue on the primary repository. Objectives receive
only the `objective` label — no category or package labels.

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

#### 5. Update `_project/phase.md`

Create or update `_project/phase.md` with:

- Phase name and version target
- Phase scope and goals
- Objectives table (with status)
- Constraints and cross-cutting decisions

#### 6. Project Board Updates

- Add each Objective to the project board
- Assign to the appropriate phase
- Verify milestones are set on the primary repository

### Outcomes

- Version target established for the phase (used for dev pre-release tags)
- Objectives created on the primary repository with the `objective` label and `Objective` issue type
- Objectives added to the project board and assigned to the phase
- Each Objective has enough context to drive an Objective Planning session
- `_project/phase.md` created with phase scope, objectives, and version target

---

## Objective Planning

### Invocation

```
/dev-workflow plan objective <issue-number>
```

### Session Metadata

1. **Issue number** — the Objective (parent issue) to plan
2. **Target repositories** — identified from the Objective body or collected

### Workflow

#### 1. Context Gathering

- Read the Objective issue for scope, architecture context, and dependencies
- Read `_project/README.md` and `_project/phase.md` for project and phase context
- Review relevant concept documents in `.claude/context/concepts/`
- Explore the affected repositories to understand current state
- Load domain-specific skills for architecture decisions

#### 2. Architecture Discussion

- Identify the specific changes needed across each repository
- Discuss architectural approaches and trade-offs
- Define interfaces and contracts between components
- Resolve open design questions relevant to this Objective

#### 3. Sub-Issue Decomposition

- Break the Objective into individual development tasks
- Each sub-issue should be resolvable in a single development session
- Each sub-issue maps to exactly **one branch and one PR** on its repository
- Order sub-issues by dependency where possible

#### 4. Sub-Issue Creation

Create sub-issues on their respective repositories and link to the Objective. Sub-issues
receive category label(s) and package label(s) — not the `objective` label.

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
)" --json url --jq '.url')

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
  -f query='mutation($parentId: ID!, $childUrl: URI!) {
    addSubIssue(input: {issueId: $parentId, subIssueUrl: $childUrl}) {
      subIssue { number title url }
    }
  }' -f parentId="$PARENT_ID" -f childUrl="$CHILD_URL"
```

#### 5. Update `_project/objective.md`

Create or update `_project/objective.md` with:

- Objective title and parent issue reference
- Phase reference
- Scope and acceptance criteria
- Sub-issues table (with status)
- Architecture decisions specific to this objective

#### 6. Project Board Updates

- Add each sub-issue to the project board
- Assign to the same phase as the parent Objective
- Verify milestones are set on each repository

### Outcomes

- Sub-issues created on their respective repositories with bootstrap context
- Sub-issues assigned the `Task` (or `Bug`) issue type and linked to the Objective via GraphQL API
- Each sub-issue has enough context to drive a Task Execution session
- Architecture decisions documented in the sub-issue bodies
- `_project/objective.md` created with scope, sub-issues, and architecture decisions
