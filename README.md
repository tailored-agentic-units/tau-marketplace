# TAU Marketplace

Plugin marketplace for the [Tailored Agentic Units](https://github.com/tailored-agentic-units) ecosystem, providing shared Claude Code skills, conventions, and tooling across TAU repositories.

## Available Plugins

| Plugin | Description |
|--------|-------------|
| [tau](./plugins/tau/) | Core plugin — development workflows, GitHub operations, Go patterns, project management, and kernel usage guides |

## Installation

```bash
claude plugin marketplace add tailored-agentic-units/tau-marketplace
claude plugin install tau@tau-marketplace
```

## Update

```bash
claude plugin marketplace update
claude plugin update tau@tau-marketplace
```

## Remove

```bash
claude plugin remove tau@tau-marketplace
claude plugin marketplace remove tailored-agentic-units/tau-marketplace
```

## Configuration

### 1. Configure skill permissions

Plugin skills are namespaced with `tau:`. Add the skills your project needs to the `permissions.allow` array:

```json
{
  "permissions": {
    "allow": [
      "Skill(tau:dev-workflow)",
      "Skill(tau:github-cli)",
      "Skill(tau:go-patterns)",
      "Skill(tau:kernel)",
      "Skill(tau:project-management)"
    ]
  }
}
```

Only include the skills relevant to your project. For example, a repository that doesn't use the TAU kernel would omit `Skill(tau:kernel)`.

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
      "Skill(tau:dev-workflow)",
      "Skill(tau:github-cli)",
      "Skill(tau:go-patterns)",
      "Skill(tau:kernel)",
      "Skill(tau:project-management)"
    ]
  }
}
```

## How It Works

Skills load automatically based on conversational context. When Claude detects relevant triggers (e.g., discussing Go interface design, creating a GitHub release, or managing project boards), it loads the appropriate skill to provide specialized guidance and commands.

User-invocable skills can also be triggered directly with slash commands using the `tau:` namespace prefix (e.g., `/tau:dev-workflow`, `/tau:github-cli`).

See the [tau plugin README](./plugins/tau/) for a detailed breakdown of available skills and their capabilities.

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
├── CHANGELOG.md                   # Release notes
├── plugins/
│   └── tau/                       # Core TAU plugin
│       ├── .claude-plugin/
│       │   └── plugin.json        # Plugin manifest (version source of truth)
│       ├── .lsp.json              # Language server configuration
│       └── skills/                # 6 skills (see plugin README)
└── README.md
```
