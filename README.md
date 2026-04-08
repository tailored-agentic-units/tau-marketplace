# TAU Marketplace

Plugin marketplace for the [Tailored Agentic Units](https://github.com/tailored-agentic-units) ecosystem, providing shared Claude Code skills, conventions, and tooling across TAU repositories.

## Available Plugins

| Plugin | Description |
|--------|-------------|
| [dev-workflow](./plugins/dev-workflow/) | Structured development sessions: concept, planning, task execution, review, release |
| [github-cli](./plugins/github-cli/) | GitHub CLI operations: issues, PRs, releases, labels, secrets, sub-issues |
| [go-patterns](./plugins/go-patterns/) | Go design patterns: interfaces, error handling, package structure, configuration |
| [project-management](./plugins/project-management/) | GitHub Projects v2: project boards, phases, objectives, cross-repo backlog |
| [iterative-dev](./plugins/iterative-dev/) | Lightweight iterative development sessions with issue-driven lifecycle |



## Installation

```bash
claude plugin marketplace add tailored-agentic-units/tau-marketplace

# Install the plugins you need
claude plugin install dev-workflow@tau-marketplace
claude plugin install github-cli@tau-marketplace
claude plugin install go-patterns@tau-marketplace
claude plugin install project-management@tau-marketplace
claude plugin install iterative-dev@tau-marketplace

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

## Migrating from the monolithic `tau` plugin

If you have the old `tau` plugin installed, remove it before installing standalone plugins:

```bash
claude plugin remove tau@tau-marketplace
```

Then update any `.claude/settings.json` permissions — replace `Skill(tau:skill-name)` with `Skill(plugin:skill)` (e.g., `Skill(tau:dev-workflow)` becomes `Skill(dev-workflow:dev-workflow)`).

Remove the cached marketplace plugin install:

```bash
rm -rf ~/.claude/plugins/marketplaces/tau-marketplace
```

Remove any of the following from `~/.claude/settings.json`:

```json
{
  "enabledPlugins": {
    "<plugin>@tau-marketplace": true // remove any tau-marketplace plugins
  },
  "extraKnownMarketplaces": {
    "tau-marketplace": { // remove this whole object
      "source": {
        "source": "github",
        "repo": "tailored-agentic-units/tau-marketplace"
      }
    }
  }
}
```

Follow the [Installation](#installation) instructions above to re-install the marketplace and install any of the skills that you use.

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
      "Skill(iterative-dev:iterative-dev)",
      "Skill(project-management:project-management)"
    ]
  }
}
```

Only include the skills relevant to your project.

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
      "Skill(iterative-dev:iterative-dev)",
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
│   ├── iterative-dev/             # Lightweight iterative development
│   ├── project-management/        # GitHub Projects v2

└── README.md
```
