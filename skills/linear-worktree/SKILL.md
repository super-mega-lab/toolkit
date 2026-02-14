---
name: linear-worktree
description: Create a git worktree from a Linear issue. Use when asked to "create a worktree for TEAM-123", "worktree for TEAM-123", or when "worktree" is mentioned with a Linear issue. Creates an isolated working directory with its own branch, ideal for running multiple parallel coding sessions. Accepts both issue identifiers (TEAM-123) and full URLs.
argument-hint: <issue-id or url>
---

# Linear Worktree Skill

Create a git worktree with its own branch from a Linear issue. Each worktree is an isolated working directory, allowing multiple coding sessions to run in parallel without conflicts.

## Trigger

This skill activates when:
- The user asks to create a worktree for a Linear issue (e.g., "create a worktree for TEAM-123")
- The user mentions "worktree" alongside a Linear issue identifier or URL
- The user runs `/linear-worktree <issue>` (slash command)

Accepted input formats:
- Issue identifier: `TEAM-123`
- Full URL: `https://linear.app/myteam/issue/TEAM-123/queue-marks-items-as-failed`
- Full URL without slug: `https://linear.app/myteam/issue/TEAM-123`

The input can come from skill arguments or from the user's message directly.

Examples:
- `/linear-worktree TEAM-123`
- `/linear-worktree https://linear.app/myteam/issue/TEAM-123/queue-marks-items-as-failed`
- "create a worktree for TEAM-123"

## Workflow

### Step 1: Check Prerequisites

1. **Linear MCP**: Call `Linear:list_teams`. If the tool is unavailable or the call fails, stop with: "Linear MCP server is required but not available. Ensure the Linear MCP server is configured in your agent's MCP settings."
2. **Git**: Run `git rev-parse --git-dir`. If it fails, stop and inform the user this must be run from a git repository.

### Step 2: Parse Issue ID

Extract the issue identifier from the input. The issue ID matches the pattern `[A-Z]+-[0-9]+`.

- If input is an identifier like `TEAM-123`, use it directly
- If input is a URL like `https://linear.app/myteam/issue/TEAM-123/...`, extract the segment after `/issue/`

### Step 3: Verify Git State

Before any git operations, check for uncommitted changes:

```bash
git status --porcelain
```

**If output is non-empty (uncommitted changes exist):**
- STOP immediately
- Warn the user: "You have uncommitted changes. Please commit or stash them before creating a new branch."
- Do NOT proceed with any git operations
- Do NOT automatically stash (user maintains explicit control)

### Step 4: Fetch Issue Info

Use the Linear MCP tool to get issue details:

```
Linear:get_issue(id: "<issue-id>")
```

Extract:
- `title` - The issue title for branch naming
- `identifier` - The issue ID (e.g., "TEAM-123")

### Step 5: Branch Naming

Format: `<ISSUE-ID>/<slugified-title>`

Rules:
1. Issue ID in UPPERCASE: `TEAM-123`
2. Title slugified:
   - Convert to lowercase
   - Replace spaces and special characters with hyphens
   - Remove consecutive hyphens
   - Trim to ~50 characters max (break at word boundary)
   - Remove leading/trailing hyphens

Examples:
| Issue ID | Title | Branch Name |
|----------|-------|-------------|
| TEAM-123 | Queue marks items as failed | `TEAM-123/queue-marks-items-as-failed` |
| TEAM-456 | Add user authentication flow | `TEAM-456/add-user-authentication-flow` |
| TEAM-7 | Fix bug in API endpoint for /users/:id | `TEAM-7/fix-bug-in-api-endpoint-for-users-id` |

### Step 6: Detect Remote and Default Branch

```bash
# Detect primary remote (usually "origin")
git remote | head -1

# Detect default branch
git symbolic-ref refs/remotes/<remote>/HEAD | sed 's|refs/remotes/<remote>/||'
```

If `symbolic-ref` fails, try `main` then `master` as fallbacks.

### Step 7: Select Worktree Directory

Determine where to place the worktree. Check in order:

1. **Existing convention:** Look for `.worktrees/` or `worktrees/` directory in the project root
2. **Ask the user** if no existing convention is found:
   ```
   Where should the worktree be created?

   1. .worktrees/ (project-local, alongside repo)
   2. A custom path
   ```

#### Safety Verification (project-local only)

If using a project-local directory (`.worktrees/` or `worktrees/`), verify it is gitignored:

```bash
git check-ignore .worktrees/
```

If NOT ignored:
- Warn the user that the worktree directory is not in `.gitignore`
- Ask if they want to add it (recommended) before proceeding
- If they agree: add the directory to `.gitignore`, stage, and commit

### Step 8: Create Worktree

```bash
# Fetch latest
git fetch <remote>

# Create the worktree with a new branch from the default branch
git worktree add <worktree-path>/<branch-name> -b <branch-name> <remote>/<default-branch>
```

Where `<worktree-path>` is the resolved directory from Step 7 and `<branch-name>` uses the naming convention from Step 5.

### Step 9: Project Setup

Inside the new worktree directory, auto-detect and run project setup:

- `package.json` exists → `npm install` (or `yarn`/`pnpm` based on lockfile)
- `Cargo.toml` exists → `cargo build`
- `requirements.txt` / `pyproject.toml` exists → `pip install -r requirements.txt` or equivalent
- `go.mod` exists → `go mod download`

Skip this step if no recognizable project files are found.

### Step 10: Baseline Test

Run the project's test suite inside the worktree:

```bash
# Auto-detect test command (npm test, cargo test, pytest, go test, etc.)
```

Report results to the user. If tests fail, ask:

```
Baseline tests have failures (<N> failing). These failures exist before your changes.
Should I continue, or would you like to investigate first?
```

### Step 11: Confirm

After successful worktree creation:

```
Worktree ready at <full-path>
Branch: <branch-name>
Tests: <N> passing

Ready to work on: [Issue Title]
```

### Step 12: Set Ticket to In Progress

After successfully creating the worktree, update the Linear issue status to "In Progress":

1. List the team's workflow states with `Linear:list_issue_statuses` to find the "In Progress" (or equivalent) state ID
2. If the issue is already "In Progress" or further along (e.g., "In Review"), do not change the status
3. Otherwise, call `Linear:update_issue` with the issue ID and the "In Progress" state ID

## Error Handling

| Scenario | Action |
|----------|--------|
| Linear MCP unavailable | Stop with setup instructions (see Step 1) |
| Not a git repository | Stop and inform user (see Step 1) |
| Uncommitted changes | Stop and warn user to commit/stash first |
| Linear API error | Report the error, do not proceed |
| Issue not found | Report "Issue not found" error |
| Branch already exists | Inform user, ask if they want to use the existing worktree or create a new one |
| Default branch detection fails | Try `main` then `master` as fallbacks, otherwise report error |
| Worktree already exists for branch | Inform user, ask if they want to use the existing worktree |
| Worktree directory not writable | Report permission error, suggest alternative directory |
| Project setup fails | Warn user but continue (setup failure is non-blocking) |
| Baseline tests fail | Report failures, ask user whether to continue or investigate |
