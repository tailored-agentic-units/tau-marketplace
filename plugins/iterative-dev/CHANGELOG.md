# Changelog

## [v0.1.2]
- Remove "Set Issue Status" step from init command — issue labels are the wrong mechanism for session status in a project-agnostic skill (commands/init.md)

## [v0.1.1]
- Enumerate AI-owned surfaces in Role Boundaries (SKILL.md)
- Fix guide conventions to prevent doc comments and integration commentary from leaking into guide code blocks (SKILL.md, commands/init.md)
- Frame guide-to-closeout documentation handoff explicitly (commands/init.md)
- Broaden "Reconcile Project Docs" to cover full doc surface: project docs, repo README, CLAUDE.md, memory files (commands/init.md)

## [v0.1.0]
- Initial release: lightweight iterative development workflow with issue-driven sessions
- Role boundaries: developer owns source code, AI owns testing/docs/closeout
- Implementation guides at `.claude/context/guides/`
- Commands: bootstrap, init, review
