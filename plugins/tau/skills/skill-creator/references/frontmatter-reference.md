# SKILL.md Frontmatter Reference

Complete reference for all YAML frontmatter fields in Claude Code skills.

## Field Reference

### `name`

Display name for the skill. Becomes the `/slash-command`. If omitted, uses the directory name.

- **Type**: string
- **Required**: No (defaults to directory name)
- **Constraints**: Lowercase letters, numbers, hyphens only. Max 64 characters.

```yaml
name: my-skill-name
```

### `description`

What the skill does and when to use it. Claude uses this to decide when to automatically load the skill. This is the most important field.

- **Type**: string
- **Required**: Recommended

**Pattern for effective descriptions:**

```yaml
description: >
  REQUIRED for <scope statement>.
  Use when the user asks to "<phrase 1>", "<phrase 2>", "<phrase 3>".
  Triggers: keyword1, keyword2, keyword3.

  When this skill is invoked, <explicit behavioral instructions>.
```

### `disable-model-invocation`

Prevents Claude from automatically loading this skill. Users must invoke with `/name`.

- **Type**: boolean
- **Default**: `false`

```yaml
disable-model-invocation: true
```

**Use for**: Deployments, destructive operations, workflows with side effects.

### `user-invocable`

Controls whether the skill appears in the `/` autocomplete menu.

- **Type**: boolean
- **Default**: `true`

```yaml
user-invocable: false
```

**Use for**: Background knowledge skills that Claude should use automatically but users should not invoke directly.

### `allowed-tools`

Tools Claude can use without asking permission when this skill is active.

- **Type**: string (comma-separated) or list (YAML array)

**String format** (comma-separated):

```yaml
allowed-tools: Read, Grep, Glob
```

**List format** (YAML array):

```yaml
allowed-tools:
  - "Bash(gh project list*)"
  - "Bash(gh project view*)"
  - "Bash(gh issue list*)"
  - "Read"
  - "Grep"
```

The list format is preferred when you have tool patterns with special characters or many entries.

### `context`

Set to `fork` to run in an isolated subagent context. The skill content becomes the subagent's prompt. The subagent does not have access to conversation history.

- **Type**: string
- **Values**: `fork`

```yaml
context: fork
```

**Use for**: Research tasks, isolated operations, parallel work.

### `agent`

Which subagent type to use when `context: fork` is set. Options include built-in agents or custom agents from `.claude/agents/`.

- **Type**: string
- **Values**: `Explore`, `Plan`, `general-purpose`, or custom agent name

```yaml
context: fork
agent: Explore
```

### `argument-hint`

Hint shown during autocomplete to indicate expected arguments.

- **Type**: string

```yaml
argument-hint: "[issue-number]"
argument-hint: "[filename] [format]"
```

### `model`

Model to use when this skill is active.

- **Type**: string

```yaml
model: claude-sonnet-4-20250514
```

### `hooks`

Hooks scoped to this skill's lifecycle. Hooks run shell commands at specific points during skill execution. They can be used for validation, logging, or side effects.

- **Type**: object
- **Supported hooks**: `PreToolUse`, `PostToolUse`

```yaml
hooks:
  PreToolUse:
    - matcher: Bash
      command: ./validate.sh
  PostToolUse:
    - matcher: Write
      command: ./post-write-check.sh
```

Each hook entry has:
- `matcher`: Tool name to match (e.g., `Bash`, `Write`, `Read`)
- `command`: Shell command to run when the hook triggers

String substitutions (e.g., `{{skill_name}}`) are available in hook commands.

## Complete Example

```yaml
---
name: deploy-staging
description: >
  Deploy application to staging environment.
  Use when the user asks to "deploy to staging", "push to staging",
  or "release to staging". Triggers: deploy, staging, release.

  Run the full deployment pipeline: tests, build, push, verify.
disable-model-invocation: true
allowed-tools:
  - "Bash(npm *)"
  - "Bash(docker *)"
argument-hint: "[version-tag]"
hooks:
  PreToolUse:
    - matcher: Bash
      command: ./pre-deploy-check.sh
---
```

## Combination Patterns

| Pattern | Frontmatter | Use Case |
|---------|-------------|----------|
| Auto-loaded reference | (defaults only) | Style guides, conventions |
| Manual workflow | `disable-model-invocation: true` | Deploy, commit, send |
| Background knowledge | `user-invocable: false` | System context, legacy docs |
| Isolated research | `context: fork`, `agent: Explore` | Code exploration |
| Restricted tools | `allowed-tools: Read, Grep` | Read-only operations |
| Hooked validation | `hooks: PreToolUse: ...` | Pre-execution checks |
