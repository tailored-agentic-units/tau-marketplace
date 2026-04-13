# Apply iterative-dev skill clarifications

## Context

During a signal-lab session (issue #8), the implementation guide's code blocks included godoc comments and integration commentary that the developer transcribed verbatim into source files. The skill text permits this reading — "Include: meaningful comments that clarify intent or integration context" is ambiguous about where those comments belong (guide prose vs. code blocks). Four gaps were identified in `.claude/iterative-dev-clarifications.md`, ordered by severity.

## Changes

### Gap 1: Enumerate AI-owned surfaces — `SKILL.md` lines 55-59

Replace the abstract "Role Boundaries" section with an explicit enumeration of what the developer owns vs. what the AI owns. Key additions: source code is written *without* doc comments (added at closeout), and the AI-owned surface is itemized (tests, documentation, source comments, project-management artifacts, implementation guides).

### Gap 2 (highest impact): Fix "meaningful comments" rule — `SKILL.md` lines 65-69 and `commands/init.md` lines 102-109

Replace guide conventions in both locations. Key changes:
- Remove the "Include: meaningful comments" bullet that caused the bug
- New files described as "structural skeleton without doc comments" instead of "complete implementation"
- Explicit "Exclude entirely from guide code blocks" list
- Commentary belongs in guide prose between code blocks, not inside code blocks
- Add litmus test: if a line would survive `git blame` in 6 months, it's source code; if it's only useful while reading the guide, it stays as guide prose

Both files contain this content — update both to stay in sync.

### Gap 3: Frame the guide-to-closeout handoff — `commands/init.md` lines 129-132

Replace the minimal "Documentation" section with text that explicitly frames this as the moment doc comments enter source files, deliberately deferred from the guide. Add rule against decision-history comments in source.

### Gap 4: Broaden "Reconcile Project Docs" — `commands/init.md` lines 136-143

Replace with "Reconcile Documentation" that covers the full doc surface: project docs (`.claude/project/` or equivalent), repo root `README.md`, `CLAUDE.md`, and memory files.

## Files modified

| File | Gaps |
|------|------|
| `plugins/iterative-dev/skills/iterative-dev/SKILL.md` | 1, 2 |
| `plugins/iterative-dev/skills/iterative-dev/commands/init.md` | 2, 3, 4 |

## Housekeeping

- Update `plugins/iterative-dev/CHANGELOG.md` with a `[v0.1.1]` entry
- Bump version in `plugins/iterative-dev/.claude-plugin/plugin.json` from `0.1.0` to `0.1.1`
- Delete `.claude/iterative-dev-clarifications.md` after applying (consumed)

## Release

After committing, tag and release v0.1.1 for the iterative-dev plugin using the `tau-marketplace-dev` skill.

## Verification

- Read both modified files end-to-end to confirm coherence
- Confirm no duplicate or contradictory guidance between `SKILL.md` and `commands/init.md`
- The "guide conventions" content should appear in both files and match
