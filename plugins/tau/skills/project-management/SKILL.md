---
name: project-management
description: >
  REQUIRED for GitHub Projects v2 operations including project boards, phases,
  and cross-repo backlog management via the gh CLI.
  Use when the user asks to "create a project", "list projects", "add phase",
  "assign phase", "view backlog", "bootstrap project", "link repo to project",
  "move items between phases", or any gh project operation.
  Triggers: project, phase, objective, backlog, board, project management, milestone,
  roadmap, cross-repo, project item, project field, sub-issue, versioning, dev release.

  When this skill is invoked, use the gh project CLI to execute the requested
  operation. Always use --format json with --jq for structured output when
  parsing IDs. For composite workflows, chain commands to resolve IDs.
allowed-tools:
  - "Bash(gh project list*)"
  - "Bash(gh project view*)"
  - "Bash(gh project field-list*)"
  - "Bash(gh project item-list*)"
  - "Bash(gh issue list*)"
  - "Bash(gh issue view*)"
  - "Bash(gh auth status*)"
user-invocable: true
---

# GitHub Project Management

Manage GitHub Projects v2, phases, and cross-repo backlogs using the `gh project` CLI.

## When This Skill Applies

**ALWAYS invoke this skill when the user's request involves ANY of these:**

- Creating, listing, viewing, editing, closing, or deleting **projects**
- Linking or unlinking **repositories** to/from a project
- Creating or managing **phase fields** (SINGLE_SELECT fields on a project)
- Creating or managing **objectives** (parent issues with cross-repo sub-issues)
- Adding issues/PRs to a project or managing **backlog items**
- Viewing or filtering items **by phase** or **by objective**
- **Bootstrapping** a new project with repos and phases
- Moving items **between phases** or viewing **phase progress**
- Any **cross-repo** project board operation

> **Scope boundary**: This skill covers `gh project *` commands. For issue/PR CRUD,
> releases, search, and actions, use the **tau:github-cli** skill instead.

## Concepts

### Four-Tier Hierarchy

| Tier | GitHub Construct | Scope | Purpose |
|------|-----------------|-------|---------|
| **Project** | GitHub Projects v2 | Org or user | Cross-repo board tracking a holistic project |
| **Phase** | SINGLE_SELECT custom field | Per project | Sequential development efforts toward the long-term vision |
| **Objective** | Parent issue with sub-issues | Cross-repo | Aggregates related issues within a phase subsection |
| **Issue** | Repository issue | Per repo | Individual development task (one branch, one PR) |

### Phase as Cross-Repo Milestone

Phases use a SINGLE_SELECT field on the project to group items from any linked repo.
Example phases: "Backlog", "Phase 1 - Foundation", "Done".

### Objective Convention

An Objective is a parent issue that aggregates sub-issues across one or more repositories.
Objectives represent a cohesive set of requirements within a phase — they are the bridge
between high-level phase goals and individual development tasks.

- Created on the **primary repository** with the `objective` label
- Sub-issues are created on their **respective repositories** and linked via GraphQL API
- Each sub-issue maps to exactly **one branch and one PR** on its repository
- Development sessions focus on resolving **one sub-issue at a time**

### Planning Lifecycle

The four-tier hierarchy is populated through a top-down planning process:

```
Concept Development  →  creates phases, repos, project infrastructure
Phase Planning       →  decomposes a phase into Objectives
Objective Planning   →  decomposes an Objective into sub-issues (architecture decisions here)
Task Execution       →  resolves a single sub-issue (one branch, one PR)
```

- **Phase Planning** produces Objectives on the primary repository
- **Objective Planning** produces sub-issues across repos, irons out architecture details
- **Task Execution** resolves one sub-issue per session

See the **tau:dev-workflow** skill for planning session workflows.
See [references/objective-management.md](references/objective-management.md) for GraphQL commands and patterns.

### Versioning Convention

Phases correspond to semantic version targets. Pre-release tags track progress during development.

| Concept | Pattern | Example |
|---------|---------|---------|
| Phase target | `v<major>.<minor>.<patch>` | `v0.1.0` |
| Dev pre-release | `v<phase-target>-dev.<NN>` | `v0.1.0-dev.01` |
| Baseline | `v0.0.x` | `v0.0.2` (initial setup, pre-phase) |

**Dev release lifecycle:**

- Each merged PR triggers a dev version bump for the modules that changed
- Dev numbers are sequential per module per phase (e.g., `dev.01`, `dev.02`, `dev.03`)
- Only **current + previous** dev releases are retained; older dev releases are deleted
- When a downstream dependency gets a new dev tag, upstream modules update their `go.mod` and are re-tagged in the same operation (dependency cascade)

**Phase release:**

- When all objectives in a phase are complete, the clean version is released (e.g., `v0.1.0`)
- All remaining dev releases for that phase are deleted
- CHANGELOG entries accumulated under `## Current` are converted to the versioned entry

**Go module proxy note:** The Go module proxy (`proxy.golang.org`) caches versions permanently once fetched. Deleted dev tags may persist in the proxy cache but won't cause issues since no `go.mod` will reference them after cascade updates.

See the **tau:dev-workflow** release reference for tagging workflows.

### Milestone Convention

Each non-meta phase (i.e., not "Backlog" or "Done") gets a **corresponding milestone**
on every linked repository. Milestone names match phase names exactly.

| Construct | Scope | Purpose |
|-----------|-------|---------|
| **Phase** | Cross-repo (project) | Organize items across all repos |
| **Milestone** | Per-repo | Progress tracking within a single repo |

### ID Chain

Many operations require resolved IDs. The chain flows:

```
project number --> project ID       (gh project view --format json)
                   field name --> field ID     (gh project field-list --format json)
                                  option name --> option ID  (from field-list options)
item URL -------> item ID           (gh project item-add --format json)
```

All IDs are opaque strings (e.g., `PVT_...`, `PVTSSF_...`, `PVTSO_...`, `PVTI_...`).
See [references/id-resolution.md](references/id-resolution.md) for detailed patterns.

## Label Convention

All TAU Platform repositories use a shared label taxonomy focused on work type categorization.

### Standard Labels

| Label | Description | Color |
|-------|-------------|-------|
| `bug` | Something isn't working correctly | `d73a4a` |
| `feature` | New capability or functionality | `0075ca` |
| `improvement` | Enhancement to existing functionality | `a2eeef` |
| `refactor` | Code restructuring without behavior change | `d4c5f9` |
| `documentation` | Documentation additions or updates | `0e8a16` |
| `testing` | Test additions or improvements | `fbca04` |
| `infrastructure` | CI/CD, build, tooling, project setup | `e4e669` |
| `objective` | Parent issue aggregating cross-repo sub-issues | `5319e7` |

See the bootstrap and clone label commands in [references/project-lifecycle.md](references/project-lifecycle.md).

## Command Reference

| Category | Reference | Key Commands |
|----------|-----------|--------------|
| Project CRUD | [project-lifecycle.md](references/project-lifecycle.md) | create, list, view, edit, close, delete, link/unlink repos |
| Phase fields | [phase-management.md](references/phase-management.md) | field-create, field-list, field-delete, option management |
| Backlog items | [backlog-management.md](references/backlog-management.md) | item-add, item-create, item-list, item-edit, archive |
| Objectives | [objective-management.md](references/objective-management.md) | create objective, add/list/remove sub-issues, GraphQL API |
| Multi-step flows | [composite-workflows.md](references/composite-workflows.md) | bootstrap project, bulk ops, progress views |
| ID resolution | [id-resolution.md](references/id-resolution.md) | project ID, field ID, option ID, item ID patterns |

## Best Practices

1. **Structured output**: Always use `--format json` with `--jq` when extracting IDs or filtering items programmatically
2. **Cache field data**: Fetch `field-list` once and extract multiple values with `jq` rather than making repeated API calls
3. **Limit item-list**: Use `-L` flag to set appropriate limits; default is 30, which may miss items in larger projects
4. **ID opacity**: Never hardcode IDs across sessions; always resolve them fresh since project items can be reindexed
5. **Phase field convention**: Name the field "Phase" consistently across all projects for predictable `--jq` selectors
6. **Scope boundary**: Use this skill for `gh project *` commands. For issue/PR lifecycle (create, edit, close, label, assign), use the **tau:github-cli** skill
7. **Token scope**: `gh project` commands require the `project` scope. Verify with `gh auth status` and add with `gh auth refresh -s project`
8. **Web fallback**: Use `--web` on `gh project view` or `gh project list` to open in browser when detailed visualization is needed
9. **Draft items**: Use `item-create` for quick backlog entries that don't yet need a full issue in a specific repository
10. **Archive over delete**: Prefer `item-archive` over `item-delete` to preserve history

## Error Handling

### Common Errors and Resolutions

| Error | Cause | Resolution |
|-------|-------|------------|
| `NOT_FOUND` | Invalid project number, owner, or item ID | Verify the project number with `gh project list --owner <owner>`. Re-resolve IDs -- they are opaque and may change. |
| `INSUFFICIENT_SCOPES` | Missing `project` OAuth scope | Run `gh auth refresh -s project` to add the required scope, then retry. |
| Field type mismatch | Using `--text` on a SINGLE_SELECT field or vice versa | Check the field type with `gh project field-list` and use the correct flag (`--single-select-option-id` for SINGLE_SELECT, `--text` for TEXT, `--number` for NUMBER). |
| `Could not resolve to a ProjectV2` | Owner name is wrong or project was deleted | Confirm the owner with `gh project list --owner <owner>`. Projects are scoped to org or user. |
| Empty `--jq` result | No matching items or incorrect jq selector | Run the command without `--jq` first to inspect the raw JSON structure, then adjust the selector. |
