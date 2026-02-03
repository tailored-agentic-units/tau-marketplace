# TAU Plugin

Core Claude Code plugin for the Tailored Agentic Units ecosystem. Provides shared skills for development workflows, GitHub operations, Go design patterns, project management, and library usage guides.

## Skills

### Background Knowledge

| Skill | Description |
|-------|-------------|
| tau-overview | Automatically loaded plugin context: skill catalog, cross-skill integration map, context document conventions, session continuity patterns. Not directly invocable. |

### Development Workflow

| Skill | Description |
|-------|-------------|
| [dev-workflow](./skills/dev-workflow/) | Structured development sessions with four modes: concept development, task execution, project review, and release. Orchestrates other skills based on session type. |

### GitHub Operations

| Skill | Description |
|-------|-------------|
| [github-cli](./skills/github-cli/) | GitHub repository operations via `gh` CLI — issues, pull requests, releases, labels, discussions, secrets, variables, workflows, gists, and API access. |
| [project-management](./skills/project-management/) | GitHub Projects v2 — project boards, phase management, backlog operations, and composite workflows. Read-only `gh` operations; write operations require manual review. |

### Code Patterns

| Skill | Description |
|-------|-------------|
| [go-patterns](./skills/go-patterns/) | Go design patterns for the TAU ecosystem: encapsulation, interface-based layer interconnection, package dependency hierarchy, configuration lifecycle, parameter encapsulation, and modern Go idioms. |

### Library Guides

| Skill | Description |
|-------|-------------|
| [tau-core](./skills/tau-core/) | Building applications with `tau-core` — agent creation, protocol execution (Chat, Vision, Tools, Embeddings), provider configuration, response handling, and error patterns. |
| [tau-orchestrate](./skills/tau-orchestrate/) | Building applications with `tau-orchestrate` — hub coordination, messaging patterns, state graphs, workflow composition (sequential, parallel, conditional), and observability. |

### Skill Authoring

| Skill | Description |
|-------|-------------|
| [skill-creator](./skills/skill-creator/) | Creating and modifying Claude Code skills — SKILL.md structure, frontmatter conventions, reference files, allowed-tools, hooks, and string substitutions. |

## Cross-Skill Integration

The `dev-workflow` skill acts as an orchestrator, loading other skills based on session type:

| Session Type | Skills Loaded |
|-------------|---------------|
| Concept Development | project-management, github-cli |
| Task Execution | github-cli, go-patterns, skill-creator, plus dev-type references |
| Project Review | project-management, github-cli |
| Release | github-cli, plus dev-type references |

## How Skills Load

Skills activate automatically when Claude detects relevant context in the conversation. Each skill's frontmatter includes a `description` field with trigger keywords that Claude matches against. For example, discussing "interface design" or "error handling patterns" triggers `go-patterns`, while mentioning "GitHub Projects" or "phase management" triggers `project-management`.

User-invocable skills can also be triggered explicitly:

```
/tau:dev-workflow task 42
/tau:github-cli
/tau:project-management
```

## Language Server Support

The plugin includes a `.lsp.json` configuration for Go development using `gopls`:

```json
{
  "go": {
    "command": "gopls",
    "args": ["serve"],
    "extensionToLanguage": {
      ".go": "go"
    }
  }
}
```

## Naming Convention

| Location | Pattern | Example | Purpose |
|----------|---------|---------|---------|
| Plugin skill | `tau-[lib]` | `tau-core`, `tau-orchestrate` | Library usage guide (building WITH the library) |
| Repo skill | `tau-[lib]-dev` | `tau-core-dev`, `tau-orchestrate-dev` | Contributing guide (developing the library itself) |

Plugin skills ship in this marketplace. Repo-level `-dev` skills remain in their respective repositories and provide contributor-specific context (architecture, testing conventions, extension patterns).
