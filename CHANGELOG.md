# Changelog

## 0.1.1

- Fix `gh issue create` commands using unsupported `--json` output flag — capture stdout directly instead
- Fix `addSubIssue` GraphQL mutation using `subIssueUrl` (URI!) which causes type mismatch with `gh api graphql -f` — use `subIssueId` (ID!) instead
- Remove broken "Add by URL" alternative sections from sub-issue reference docs

## 0.1.0

- Split planning command into dedicated `phase` and `objective` sub-commands with transition closeout logic
- Add transition closeout workflow: close previous phase/objective, handle incomplete work (carry forward or move to backlog), enforce no-orphaned-issues principle
- Rename all command files to match their command names (concept.md, phase.md, objective.md, task.md, review.md, release.md)
- Add Phases roadmap (`_project/README.md`) and objectives table (`_project/phase.md`) as reference points for phase/objective sequencing
- Remove `_project/` document archival — GitHub project infrastructure serves as historical record
- Add remediation convention to task execution implementation guide template
- Remove project-specific language from project-agnostic commands (release, review)

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
