# Marketplace Refactor

Decompose the monolithic `tau` plugin into standalone per-skill plugins. Triggered by the library extraction effort — as tau/agent and tau/orchestrate become standalone libraries, they need their own marketplace skills, and the existing monolithic plugin structure cannot support selective installation.

## Problem

The current `tau` plugin bundles 6 skills into a single installable unit. Users must install all or nothing. As the TAU ecosystem grows (tau/agent, tau/orchestrate, tau/kernel), this creates plugin saturation — repositories that only need agent skills are forced to install kernel and orchestrate skills too.

## Current Structure

```
plugins/
└── tau/
    ├── .claude-plugin/
    │   └── plugin.json          # v0.1.3
    ├── skills/
    │   ├── dev-workflow/         # Structured development sessions
    │   ├── github-cli/           # GitHub CLI operations
    │   ├── go-patterns/          # Go design patterns
    │   ├── kernel/               # TAU kernel usage guide
    │   │   └── references/       # core.md, agent.md, orchestrate.md, kernel.md
    │   ├── project-management/   # GitHub Projects v2
    │   └── tau-overview/         # Ecosystem conventions
    └── .lsp.json
```

**marketplace.json** currently registers one plugin:
```json
{
  "plugins": [
    { "name": "tau", "source": "./plugins/tau" }
  ]
}
```

## Target Structure

Every skill becomes a standalone plugin. Each plugin has its own `.claude-plugin/plugin.json`, versioned independently.

```
plugins/
├── tau-dev-workflow/
│   ├── .claude-plugin/
│   │   └── plugin.json
│   └── skills/
│       └── dev-workflow/        # Moved from tau/skills/dev-workflow
├── tau-github-cli/
│   ├── .claude-plugin/
│   │   └── plugin.json
│   └── skills/
│       └── github-cli/          # Moved from tau/skills/github-cli
├── tau-go-patterns/
│   ├── .claude-plugin/
│   │   └── plugin.json
│   └── skills/
│       └── go-patterns/         # Moved from tau/skills/go-patterns
├── tau-project-management/
│   ├── .claude-plugin/
│   │   └── plugin.json
│   └── skills/
│       └── project-management/  # Moved from tau/skills/project-management
├── tau-overview/
│   ├── .claude-plugin/
│   │   └── plugin.json
│   └── skills/
│       └── tau-overview/        # Moved from tau/skills/tau-overview
├── tau-agent/
│   ├── .claude-plugin/
│   │   └── plugin.json
│   └── skills/
│       └── agent/               # NEW: tau/agent usage guide
├── tau-orchestrate/
│   ├── .claude-plugin/
│   │   └── plugin.json
│   └── skills/
│       └── orchestrate/         # NEW: tau/orchestrate usage guide
└── tau-kernel/
    ├── .claude-plugin/
    │   └── plugin.json
    └── skills/
        └── kernel/              # Migrated: kernel-specific content only
```

**marketplace.json** registers all plugins:
```json
{
  "plugins": [
    { "name": "tau-dev-workflow", "source": "./plugins/tau-dev-workflow" },
    { "name": "tau-github-cli", "source": "./plugins/tau-github-cli" },
    { "name": "tau-go-patterns", "source": "./plugins/tau-go-patterns" },
    { "name": "tau-project-management", "source": "./plugins/tau-project-management" },
    { "name": "tau-overview", "source": "./plugins/tau-overview" },
    { "name": "tau-agent", "source": "./plugins/tau-agent" },
    { "name": "tau-orchestrate", "source": "./plugins/tau-orchestrate" },
    { "name": "tau-kernel", "source": "./plugins/tau-kernel" }
  ]
}
```

## Skill Content Migration

### Existing Skills (direct move)

These skills move as-is into standalone plugins. No content changes required beyond updating any cross-references between skills.

| Skill | From | To |
|-------|------|-----|
| dev-workflow | `tau/skills/dev-workflow/` | `tau-dev-workflow/skills/dev-workflow/` |
| github-cli | `tau/skills/github-cli/` | `tau-github-cli/skills/github-cli/` |
| go-patterns | `tau/skills/go-patterns/` | `tau-go-patterns/skills/go-patterns/` |
| project-management | `tau/skills/project-management/` | `tau-project-management/skills/project-management/` |
| tau-overview | `tau/skills/tau-overview/` | `tau-overview/skills/tau-overview/` |

### kernel Skill (content split)

The current `tau/skills/kernel/` covers the entire kernel monolith including agent, orchestrate, and kernel-specific content. After library extraction, this content must be split:

| Reference File | Content | Destination Plugin |
|----------------|---------|-------------------|
| `references/core.md` | Config, protocol, response, model types | `tau-agent` |
| `references/agent.md` | Agent creation, protocols, providers, mocks | `tau-agent` |
| `references/orchestrate.md` | Hubs, messaging, state, workflows | `tau-orchestrate` |
| `references/kernel.md` | Runtime architecture, runtime loop | `tau-kernel` |

The kernel SKILL.md itself must be rewritten to focus only on the kernel harness (runtime loop, sessions, memory, tools, MCP, API). It should reference tau/agent and tau/orchestrate as dependencies.

### New Skills (to be written)

| Plugin | Skill Content |
|--------|---------------|
| `tau-agent` | tau/agent usage guide — agent creation, provider setup, format layer, streaming, registry, protocol execution, mock testing |
| `tau-orchestrate` | tau/orchestrate usage guide — hub coordination, state graphs, workflows, observability, Participant interface |

These skills should be written after their respective libraries reach v0.1.0 so the content reflects the actual API.

## Dev Skills (NOT in marketplace)

Dev skills are co-located in their repositories, not in the marketplace:

| Library | Dev Skill Location | Content |
|---------|-------------------|---------|
| tau/agent | `~/tau/agent/.claude/skills/agent-dev/` | Contributing guide: adding providers, formats, streaming readers |
| tau/orchestrate | `~/tau/orchestrate/.claude/skills/orchestrate-dev/` | Contributing guide: adding workflow patterns, observers, hub features |
| tau/kernel | `~/tau/kernel/.claude/skills/kernel-dev/` | Contributing guide (already exists, needs update post-extraction) |

## Plugin Versioning

Each plugin gets independent versioning in its `plugin.json`. Initial version for new plugins: `0.1.0`. Existing skills that are moved should start at `0.1.0` as well since this is a structural break from the monolithic plugin.

## Installation Impact

**Before** (monolithic):
```
claude plugin install tau@tau-marketplace
```

**After** (selective):
```
# For tau/agent development
claude plugin install tau-agent@tau-marketplace
claude plugin install tau-go-patterns@tau-marketplace

# For tau/kernel development
claude plugin install tau-kernel@tau-marketplace
claude plugin install tau-agent@tau-marketplace
claude plugin install tau-orchestrate@tau-marketplace

# For any TAU development
claude plugin install tau-dev-workflow@tau-marketplace
claude plugin install tau-github-cli@tau-marketplace
claude plugin install tau-project-management@tau-marketplace
```

## CHANGELOG

The marketplace CHANGELOG should document the decomposition as a single release entry (e.g., `0.2.0`) that:
- Removes the monolithic `tau` plugin
- Introduces all standalone plugins at `0.1.0`
- Notes breaking change: users must update their plugin installations

## Execution Order

1. Create standalone plugin directories for existing skills (structural move)
2. Update marketplace.json to register all new plugins
3. Remove the monolithic `tau/` plugin directory
4. Write new `tau-agent` and `tau-orchestrate` skill content (after libraries reach v0.1.0)
5. Split kernel skill content across tau-agent, tau-orchestrate, and tau-kernel
6. Update CHANGELOG and tag release

Steps 1-3 can happen early. Steps 4-5 should wait until library APIs are stable.
