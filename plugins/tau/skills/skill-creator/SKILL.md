---
name: skill-creator
description: >
  REQUIRED for creating or modifying Claude Code skills.
  Use when the user asks to "create a skill", "new skill", "add a skill",
  "modify skill", or needs help with SKILL.md format and frontmatter.
  Triggers: create skill, new skill, SKILL.md, skill template, frontmatter,
  slash command, custom command.

  When creating skills, follow this project's patterns: single SKILL.md per directory,
  YAML frontmatter with explicit trigger descriptions, "When This Skill Applies" section,
  organized content with code examples. Keep SKILL.md under 500 lines.
  After creating a skill, add it to .claude/settings.json permissions (alphabetical order).
---

# Claude Code Skill Creator

Create and maintain Claude Code skills following project conventions.

## When This Skill Applies

**ALWAYS invoke this skill when the user's request involves ANY of these:**

- Creating a new Claude Code skill
- Modifying an existing skill's SKILL.md or frontmatter
- Adding supporting files to a skill directory
- Questions about skill structure, frontmatter fields, or triggers
- Migrating `.claude/commands/` to skills format

## Skill Directory Structure

```
.claude/skills/<skill-name>/
├── SKILL.md               # Required: main instructions
└── references/            # Optional: detailed reference docs
    └── detailed-guide.md
```

Each skill is a single directory. The `SKILL.md` file is the entrypoint. Supporting files in `references/` are loaded on demand when referenced from SKILL.md.

## Creating a New Skill

### Step 1: Create the Directory

```bash
mkdir -p .claude/skills/<skill-name>
```

Use kebab-case for directory names (e.g., `github-cli`, `skill-creator`).

### Step 2: Write SKILL.md

Every SKILL.md has two parts:

1. **YAML frontmatter** (between `---` markers) - metadata and triggers
2. **Markdown content** - instructions Claude follows when the skill is invoked

```yaml
---
name: my-skill
description: >
  REQUIRED for <specific context>.
  Use when the user asks to "<action 1>", "<action 2>", "<action 3>".
  Triggers: keyword1, keyword2, keyword3.

  When this skill is invoked, <explicit instructions for what to do>.
---

# Skill Title

## When This Skill Applies

**ALWAYS invoke this skill when the user's request involves ANY of these:**

- Specific scenario 1
- Specific scenario 2
- Specific scenario 3

## Section 1

Content with code examples...

## Section 2

More content...
```

### Step 3: Add to Settings

Add the new skill to `.claude/settings.json` permissions in **alphabetical order**:

```json
{
  "permissions": {
    "allow": [
      "Skill(existing-skill)",
      "Skill(new-skill)",
      "Skill(other-skill)"
    ]
  }
}
```

## Frontmatter Reference

See [references/frontmatter-reference.md](references/frontmatter-reference.md) for complete field documentation.

### Essential Fields

| Field | Purpose |
|-------|---------|
| `name` | Skill identifier, kebab-case (max 64 chars) |
| `description` | When to use + triggers + inline instructions |

### Optional Fields

| Field | Purpose |
|-------|---------|
| `disable-model-invocation` | Set `true` to prevent auto-loading (user `/name` only) |
| `user-invocable` | Set `false` to hide from `/` menu (background knowledge) |
| `allowed-tools` | Pre-approve tools (e.g., `Bash(gh *)`) |
| `context` | Set `fork` to run in an isolated subagent |
| `agent` | Subagent type when `context: fork` (e.g., `Explore`, `Plan`) |
| `argument-hint` | Shown during autocomplete (e.g., `[issue-number]`) |
| `hooks` | Lifecycle hooks scoped to this skill (e.g., `PreToolUse`, `PostToolUse`) |

## Content Best Practices

### Description Field

Write explicit, directive descriptions. Include:
- **REQUIRED for** statement declaring the scope
- **Use when** with exact trigger phrases in quotes
- **Triggers:** comma-separated keywords
- **Inline instructions** for what to do when invoked

```yaml
# Good: Explicit and directive
description: >
  REQUIRED for ANY GitHub repository operations via the gh CLI.
  Use when the user asks to "create an issue", "list issues", "create PR".
  Triggers: issue, PR, pull request, release, GitHub.

  When this skill is invoked, use the gh CLI to execute the requested operation.

# Bad: Vague
description: Helps with GitHub stuff
```

### Markdown Content

1. **Start with "When This Skill Applies"** - bulleted list of trigger scenarios
2. **Organize by topic** using H2 (`##`) and H3 (`###`) headings
3. **Include code examples** in fenced blocks with language hints
4. **Add reference tables** for quick lookup
5. **Keep under 500 lines** - move detail to `references/` files

### Supporting Files

Reference them from SKILL.md so Claude knows when to load them:

```markdown
## Additional Resources

- For complete API details, see [reference.md](references/reference.md)
```

## Project Patterns

This project's existing skills follow these conventions:

| Pattern | Example |
|---------|---------|
| Single SKILL.md per skill | `.claude/skills/go-patterns/SKILL.md` |
| YAML frontmatter with triggers | `Triggers: interface design, error handling...` |
| "When This Skill Applies" section | Bulleted list of applicable scenarios |
| Topic-based organization | Numbered principles, operational categories |
| Code examples | Inline and fenced Go/bash blocks |
| Reference tables | Package responsibilities, command references |

## String Substitutions

Skills support dynamic string substitutions in the `allowed-tools` and `hooks` fields of the frontmatter. These variables are replaced with their values at invocation time.

### Available Variables

| Variable | Replaced With | Available In |
|----------|---------------|--------------|
| `{{skill_name}}` | The skill's `name` field value | `allowed-tools`, `hooks` |
| `$ARGUMENTS` | All arguments passed with `/skill-name args` | Skill body content |
| `$ARGUMENTS[N]` or `$N` | Specific argument by index (0-based) | Skill body content |
| `${CLAUDE_SESSION_ID}` | Current session ID | Skill body content |

### Shell Substitution

An exclamation mark immediately followed by a command wrapped in backticks injects the output of that shell command before the skill runs. There must be no space between the exclamation mark and the opening backtick. For example, to inject the current git branch, write the exclamation mark then `git branch --show-current` in backticks.

## Invocation Control

| Setting | You invoke | Claude invokes | Use for |
|---------|-----------|---------------|---------|
| Default | Yes | Yes | Reference knowledge, common operations |
| `disable-model-invocation: true` | Yes | No | Destructive operations, deployments |
| `user-invocable: false` | No | Yes | Background knowledge, context |

## Related Skills

- **tau:github-cli** - GitHub CLI operations (issues, PRs, releases)
- **tau:project-management** - GitHub Projects v2 boards and phases
