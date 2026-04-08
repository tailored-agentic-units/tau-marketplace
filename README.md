# TAU Marketplace

Plugin marketplace for the [Tailored Agentic Units](https://github.com/tailored-agentic-units) ecosystem, providing shared Claude Code skills, conventions, and tooling across TAU repositories.

## Available Plugins

| Plugin | Description |
|--------|-------------|
| [dev-workflow](./plugins/dev-workflow/) | Structured development sessions: concept, planning, task execution, review, release |
| [github-cli](./plugins/github-cli/) | GitHub CLI operations: issues, PRs, releases, labels, secrets, sub-issues |
| [go-patterns](./plugins/go-patterns/) | Go design patterns: interfaces, error handling, package structure, configuration |
| [project-management](./plugins/project-management/) | GitHub Projects v2: project boards, phases, objectives, cross-repo backlog |
| [tau-overview](./plugins/tau-overview/) | TAU ecosystem overview and conventions |
| [kernel](./plugins/kernel/) | TAU kernel usage guide: core types, agent, orchestrate, runtime |

## Migrating from the monolithic `tau` plugin

If you have the old `tau` plugin installed, remove it before installing standalone plugins:

```bash
claude plugin remove tau@tau-marketplace
claude plugin marketplace update
```

Then update `.claude/settings.json` permissions — replace `Skill(tau:skill-name)` with `Skill(plugin:skill)` (e.g., `Skill(tau:dev-workflow)` becomes `Skill(dev-workflow:dev-workflow)`).

## Installation

```bash
claude plugin marketplace add tailored-agentic-units/tau-marketplace

# Install the plugins you need
claude plugin install dev-workflow@tau-marketplace
claude plugin install github-cli@tau-marketplace
claude plugin install go-patterns@tau-marketplace
claude plugin install project-management@tau-marketplace
claude plugin install tau-overview@tau-marketplace
claude plugin install kernel@tau-marketplace
```

## Update

```bash
claude plugin marketplace update
claude plugin update dev-workflow@tau-marketplace
```

## Remove

```bash
claude plugin remove dev-workflow@tau-marketplace
claude plugin marketplace remove tailored-agentic-units/tau-marketplace
```

## Configuration

### 1. Configure skill permissions

Add the skills your project needs to the `permissions.allow` array:

```json
{
  "permissions": {
    "allow": [
      "Skill(dev-workflow:dev-workflow)",
      "Skill(github-cli:github-cli)",
      "Skill(go-patterns:go-patterns)",
      "Skill(kernel:kernel)",
      "Skill(project-management:project-management)"
    ]
  }
}
```

Only include the skills relevant to your project. For example, a repository that doesn't use the TAU kernel would omit `Skill(kernel:kernel)`.

### 2. Configure tool permissions

Some skills use CLI tools that require their own permissions. Add the ones your workflow needs:

```json
{
  "permissions": {
    "allow": [
      "Bash(gh auth status*)",
      "Bash(gh issue list*)",
      "Bash(gh issue view*)",
      "Bash(gh project field-list*)",
      "Bash(gh project item-list*)",
      "Bash(gh project list*)",
      "Bash(gh project view*)"
    ]
  }
}
```

### Complete example

Here is a full `.claude/settings.json` for a TAU Go library project:

```json
{
  "plansDirectory": "./.claude/plans",
  "permissions": {
    "allow": [
      "Bash(gh auth status*)",
      "Bash(gh issue list*)",
      "Bash(gh issue view*)",
      "Bash(gh project field-list*)",
      "Bash(gh project item-list*)",
      "Bash(gh project list*)",
      "Bash(gh project view*)",
      "Skill(dev-workflow:dev-workflow)",
      "Skill(github-cli:github-cli)",
      "Skill(go-patterns:go-patterns)",
      "Skill(kernel:kernel)",
      "Skill(project-management:project-management)"
    ]
  }
}
```

## How It Works

Skills load automatically based on conversational context. When Claude detects relevant triggers (e.g., discussing Go interface design, creating a GitHub release, or managing project boards), it loads the appropriate skill to provide specialized guidance and commands.

User-invocable skills can also be triggered directly with slash commands (e.g., `/dev-workflow:dev-workflow`, `/github-cli:github-cli`).

## Development

This repository uses the `tau-marketplace-dev` skill for structured maintenance. Use `/tau-marketplace-dev update` to start an update session and `/tau-marketplace-dev release` to tag releases after PRs are merged.

## Repository Structure

```
tau-marketplace/
├── .claude/
│   ├── settings.json
│   └── skills/
│       └── tau-marketplace-dev/   # Repository development workflow
├── .claude-plugin/
│   └── marketplace.json           # Marketplace manifest
├── .github/
│   └── workflows/
│       └── release.yml            # Tag-triggered release automation
├── plugins/
│   ├── dev-workflow/              # Structured development sessions
│   ├── github-cli/                # GitHub CLI operations
│   ├── go-patterns/               # Go design patterns
│   ├── kernel/                    # TAU kernel usage guide
│   ├── project-management/        # GitHub Projects v2
│   └── tau-overview/              # Ecosystem overview
└── README.md
```
