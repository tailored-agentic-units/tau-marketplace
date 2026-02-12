# Changelog

## 0.0.8

- Refocus concept-development to produce Phases (areas of focus) instead of Objectives
- Move Objective creation to Phase Planning, enforcing clean top-down decomposition
- Add CHANGELOG update step to task execution closeout (Phase 8c) before PR creation
- Simplify dev release to tagging-only (CHANGELOG entry now created during task closeout)
- Add Phase verification steps to project-review
- Consolidate stale Library Guides references (tau-agent/core/orchestrate/runtime → kernel)
- Fix release.yml: remove non-existent verify job, add dynamic prefix extraction for multi-plugin support
- Add tau-marketplace-dev skill for structured marketplace maintenance (update + release sessions)
- Initialize CHANGELOG with release pipeline integration
- Update tau-overview, dev-workflow SKILL.md, and plugin README to reflect refined workflows

## 0.0.7

Initial release of the TAU marketplace plugin with skills:
- dev-workflow: Structured development sessions (concept, planning, task execution, review, release)
- github-cli: GitHub CLI operations reference
- go-patterns: Go design patterns for TAU ecosystem
- kernel: TAU kernel usage guide (consolidated from library-specific skills)
- project-management: GitHub Projects v2 management
- skill-creator: Claude Code skill authoring guide
- tau-overview: Plugin overview and conventions
