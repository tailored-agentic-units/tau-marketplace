# Project Review

Perform a comprehensive analysis of the project's current state.

## Process

### 1. Read the Authoritative Document

Read `.claude/project/README.md` and relevant sub-files — the vision, architecture, and requirements checklist.

### 2. Analyze the Actual Project State

Explore the codebase to understand what actually exists:
- What files and directories are present
- What functionality is implemented
- What the code actually does vs. what the project docs say it should do
- Git log for recent activity and trajectory

### 3. Identify Alignment Gaps

Compare what the project docs describe against what the codebase contains:

- **Requirements marked done but not implemented** — checked off but the code doesn't support it
- **Implemented but not tracked** — functionality exists but isn't reflected in requirements
- **Architectural drift** — the codebase has diverged from the documented architecture
- **Missing requirements** — capabilities clearly needed but not listed

### 4. Assess Trajectory

Look at the bigger picture:
- Is the project on track toward its vision?
- Are there patterns that suggest a better architectural approach?
- Are there opportunities to simplify or consolidate?
- What are the highest-leverage next steps?
- Are there risks or technical debt accumulating?

### 5. Produce the Review

Present findings to the user organized by:

1. **Alignment gaps** — what's out of sync between project docs and the codebase
2. **Recommendations** — concrete suggestions for improving the project's trajectory
3. **Proposed updates** — specific edits to project files (requirements, architecture, etc.)

After the user reviews and approves changes, update the relevant files in `.claude/project/` and commit.
