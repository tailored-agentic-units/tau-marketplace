# Session Init: Formalize iterative-dev Skill (A2)

## Goal

Adapt the iterative-dev skill from ~/code/revolutions/.claude/skills/iterative-dev into a marketplace skill at tau-marketplace. This creates a lightweight, less structured development workflow alternative to dev-workflow — focused on planning and executing a single session at a time with lightweight issue tracking.

## Source Material

The starting point is ~/code/revolutions/.claude/skills/iterative-dev which has:
- **SKILL.md** — main skill definition with sub-commands (bootstrap, init, review)
- **commands/bootstrap.md** — scaffold a new project
- **commands/init.md** — start a session from .claude/init.md in plan mode
- **commands/review.md** — analyze project state and alignment

## Key Adaptations Required

### Role Boundaries (from dev-workflow)

The formalized skill must maintain the same developer/AI responsibility split as dev-workflow:
- **Developer**: Owns source code implementation
- **AI assistant**: Owns testing, code comments, documentation, and contextual artifacts

The current iterative-dev skill doesn't enforce this boundary — the AI implements freely after plan alignment. The formalized version needs explicit role boundary language in the init and bootstrap commands.

### Implementation Guides

The formalized skill should use implementation guides following the same design as dev-workflow:
- Created during task execution (init command)
- Stored in `.claude/context/guides/`
- Transform plan-mode output into a persistent implementation reference
- Archived after session closeout

### Issue-Driven Session Lifecycle

Each session is tied to a GitHub issue for tracking:

**Session initialization (init command):**
1. Read `.claude/init.md` to identify the current session's issue number
2. Create a branch from main (descriptive name)
3. Set the issue status from Todo to In Progress

**Session closeout:**
1. Pre-commit review (reconcile docs, discuss next steps, write init.md)
2. Create a new GitHub issue for the next session (captured from the discussion)
3. Write `.claude/init.md` referencing the new issue
4. Commit, push, and open a PR that closes the current issue

This provides a lightweight audit trail without requiring project boards or phase hierarchies. Each merged PR closes one issue; each closeout creates the next.

### What to Preserve from iterative-dev

- **Lightweight structure**: No project boards or phase hierarchies — just `.claude/project/`, `.claude/init.md`, and GitHub issues
- **Plan-mode-driven sessions**: Enter plan mode, collaborate on approach, then execute
- **Pre-commit review cycle**: Reconcile project docs, discuss next steps, write init.md
- **Session bootstrap continuity**: `.claude/init.md` carries context between sessions
- **Branching convention**: Each session gets its own descriptive branch from main

### What to Remove

- **Simulation-specific content**: The "Reconcile Simulation State" step in init.md is project-specific to revolutions. Remove it from the generalized skill.
- **Skill replication into projects**: The bootstrap step that copies the skill into the project directory. Marketplace skills are installed via the plugin system instead.

### What to Add

- **Role boundary enforcement**: Clear language about developer vs. AI responsibilities
- **Implementation guide lifecycle**: Create guide from plan, use during execution, archive at closeout
- **Issue-driven session tracking**: Each session tied to an issue; closeout creates next issue and PR closes current
- **Alignment with marketplace plugin structure**: skill directory, SKILL.md frontmatter, plugin.json

## Plugin Structure

```
tau-iterative-dev/
├── .claude-plugin/
│   └── plugin.json          # v0.1.0
└── skills/
    └── iterative-dev/
        ├── SKILL.md          # Adapted from source with role boundaries
        └── commands/
            ├── bootstrap.md  # Adapted from source
            ├── init.md       # Adapted with implementation guides, role boundaries, issue lifecycle
            └── review.md     # Preserved as-is (already general)
```

## Definition of Done

- [ ] Skill adapted with role boundary enforcement
- [ ] Implementation guide lifecycle integrated into init command
- [ ] Issue-driven session lifecycle: init sets In Progress, closeout creates next issue, PR closes current
- [ ] Simulation-specific content removed
- [ ] Skill replication step replaced with marketplace installation
- [ ] Plugin structure created (plugin.json, SKILL.md with frontmatter)
- [ ] Marketplace registered in marketplace.json
- [ ] Tagged and released
