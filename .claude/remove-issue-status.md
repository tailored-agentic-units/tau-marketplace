# Remove "Set Issue Status" step from iterative-dev init command

## Problem

The `iterative-dev` skill's `init` sub-command instructs the AI to label the GitHub issue with `in-progress` at session start:

```bash
gh issue edit <number> --add-label "in-progress"
```

This step is incorrect for the skill's intended use. `iterative-dev` is project-agnostic and does not require a GitHub Project. In a repo that uses a GitHub Project, "In Progress" is a **Status field** on the project item — a column on the project board — not a label on the issue itself. Adding a literal `in-progress` label clutters the repo with a label that has no semantic meaning in either workflow:

- **Projects-backed repos** — the correct mechanism is a Status field update on the project item (`gh project item-edit ...`), not `gh issue edit --add-label`. Adding a label here is redundant at best and misleading at worst.
- **Plain repos** (no Project) — which is the `iterative-dev` default — there is no notion of issue status at all. The label serves no purpose and forces the user to either pre-create the label or see the `gh` call fail.

Either way, this step doesn't belong in a skill that explicitly doesn't use Projects.

## Fix

Remove the entire **Set Issue Status** subsection from `plugins/iterative-dev/skills/iterative-dev/commands/init.md`.

### File

`plugins/iterative-dev/skills/iterative-dev/commands/init.md`

### Remove these lines

```markdown
### Set Issue Status

```bash
gh issue edit <number> --add-label "in-progress"
```

```

(The subsection currently sits between "Read the Issue" and "Create a Session Branch" under Stage 1: Session Setup.)

### Resulting Stage 1 structure

After the change, Stage 1 flows as:

1. Read the Issue
2. Create a Session Branch
3. Read Project Context

No replacement is needed — branch creation already serves as the visible "this session is in flight" signal (the branch name encodes the issue number), and the PR that closes the issue at the end of the session is the canonical "done" signal.

## Notes for future iteration

If later we want to add first-class GitHub Projects support to `iterative-dev`, the right approach is a **separate, optional** Stage 1 step that:

- Detects whether the repo is linked to a Project
- Uses `gh project item-edit --id <item-id> --field-id <status-field> --single-select-option-id <in-progress-option-id>` to move the item on the board
- Skips silently if no Project is linked

That's a deliberate addition, not a default — and it belongs gated behind a project detection step rather than hard-coded into the init flow.
