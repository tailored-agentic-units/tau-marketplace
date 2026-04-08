# Session Init: Library-dev Skills (A3)

## Goal

Create [library]-dev skills for each extracted TAU library: agent, orchestrate, format, provider, protocol. Each skill lives within its host library repository (not the marketplace) and assists developers contributing to that specific library.

## Template

The kernel-dev skill at ~/tau/kernel/.claude/skills/kernel-dev/SKILL.md serves as the template. Each library-dev skill follows the same structure adapted to the library's scope.

### kernel-dev Structure (Template)

The kernel-dev skill covers:
1. **When This Skill Applies** — specific contribution scenarios
2. **Architecture** — Go module structure with strict package dependency hierarchy
3. **Package Responsibilities** — table mapping packages to responsibilities and key interfaces
4. **Extension Patterns** — prescriptive workflows for common contribution types (adding a provider, observer, workflow pattern)
5. **Testing Strategy** — co-located tests, black-box `*_test` packages, table-driven patterns, mocking

### Adaptation Per Library

Each library-dev skill should document:

| Section | Content |
|---------|---------|
| When it applies | Contribution scenarios specific to that library |
| Architecture | Package structure and dependency hierarchy within the library |
| Package responsibilities | What each package does, key interfaces and types |
| Extension patterns | How to add new implementations (e.g., new format, new provider) |
| Sub-module convention | Root interface + sub-module implementations with explicit Register() |
| Testing strategy | Testing patterns, mock utilities, integration test approach |

## Libraries and Their Scope

### agent-dev (~/tau/agent/.claude/skills/agent-dev/)
- Adding request types, client configurations
- Registry patterns, mock utilities
- Agent interface extensions

### orchestrate-dev (~/tau/orchestrate/.claude/skills/orchestrate-dev/)
- Adding workflow patterns (chain, parallel, conditional, custom)
- Hub and Participant patterns
- Observer implementations
- State graph extensions

### format-dev (~/tau/format/.claude/skills/format-dev/)
- Adding new wire formats (beyond OpenAI, Converse)
- Format interface implementation
- Tool definition handling
- Streaming chunk parsing

### provider-dev (~/tau/provider/.claude/skills/provider-dev/)
- Adding new providers (beyond Ollama, Azure, Bedrock)
- Provider interface implementation
- Authentication and identity patterns
- Streaming reader implementations (SSE, EventStream)

### protocol-dev (~/tau/protocol/.claude/skills/protocol-dev/)
- Evolving foundational types (Message, Response, Config, Model)
- Adding config fields and getter patterns
- Response content block types
- Streaming abstractions

## Process

For each library:
1. Read the library's go.mod, package structure, and key source files
2. Identify the architecture, dependency hierarchy, and extension points
3. Write the SKILL.md following the kernel-dev template structure
4. Place at `~/tau/[library]/.claude/skills/[library]-dev/SKILL.md`
5. Commit and push to the library's repository

## Execution Order

Start with the libraries that have the most extension points and contribution surface:
1. **provider-dev** — most sub-modules, clearest extension pattern (add a provider)
2. **format-dev** — similar pattern (add a format)
3. **agent-dev** — builds on provider/format understanding
4. **orchestrate-dev** — independent domain (workflows, hub, state)
5. **protocol-dev** — foundational types, least frequent contributions

## Definition of Done

- [ ] agent-dev skill created at ~/tau/agent/.claude/skills/agent-dev/SKILL.md
- [ ] orchestrate-dev skill created at ~/tau/orchestrate/.claude/skills/orchestrate-dev/SKILL.md
- [ ] format-dev skill created at ~/tau/format/.claude/skills/format-dev/SKILL.md
- [ ] provider-dev skill created at ~/tau/provider/.claude/skills/provider-dev/SKILL.md
- [ ] protocol-dev skill created at ~/tau/protocol/.claude/skills/protocol-dev/SKILL.md
- [ ] Each skill covers architecture, package responsibilities, extension patterns, and testing
- [ ] Each committed and pushed to its host library repository
