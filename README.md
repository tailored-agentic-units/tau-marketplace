# TAU Marketplace

Plugin marketplace for the [Tailored Agentic Units](https://github.com/tailored-agentic-units) ecosystem, providing shared Claude Code skills, conventions, and tooling across TAU repositories.

## Available Plugins

| Plugin | Description |
|--------|-------------|
| [tau](./plugins/tau/) | Core plugin — development workflows, GitHub operations, Go patterns, project management, and library usage guides |

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
      "Skill(tau:project-management)",
      "Skill(tau:skill-creator)",
      "Skill(tau:tau-core)",
      "Skill(tau:tau-orchestrate)"
    ]
  }
}
```

Only include the skills relevant to your project. For example, a repository that uses `tau-core` but not `tau-orchestrate` would omit `Skill(tau:tau-orchestrate)`.

### 3. Configure tool permissions

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
      "Skill(tau:project-management)",
      "Skill(tau:skill-creator)",
      "Skill(tau:tau-core)"
    ]
  }
}
```

## How It Works

Skills load automatically based on conversational context. When Claude detects relevant triggers (e.g., discussing Go interface design, creating a GitHub release, or managing project boards), it loads the appropriate skill to provide specialized guidance and commands.

User-invocable skills can also be triggered directly with slash commands using the `tau:` namespace prefix (e.g., `/tau:dev-workflow`, `/tau:github-cli`).

See the [tau plugin README](./plugins/tau/) for a detailed breakdown of available skills and their capabilities.

## Repository Structure

```
tau-marketplace/
├── .claude-plugin/
│   └── marketplace.json       # Marketplace manifest
├── plugins/
│   └── tau/                   # Core TAU plugin
│       ├── .claude-plugin/
│       │   └── plugin.json    # Plugin manifest
│       ├── .lsp.json          # Language server configuration
│       └── skills/            # 8 skills (see plugin README)
└── README.md
```
