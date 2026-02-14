---
name: linear-branch
description: Create a git feature branch from a Linear issue. Use when asked to "create a branch for TEAM-123", "branch for TEAM-123", or when a Linear ticket URL or issue identifier is provided. Accepts both issue identifiers (TEAM-123) and full URLs.
argument-hint: <issue-id or url>
---

# Linear Branch Skill

Create a git feature branch from a Linear issue.

## Trigger

This skill activates when:
- The user asks to create a branch for a Linear issue (e.g., "create a branch for TEAM-123")
- The user mentions "branch" alongside a Linear issue identifier or URL
- The user runs `/linear-branch <issue>` (slash command)

Accepted input formats:
- Issue identifier: `TEAM-123`
- Full URL: `https://linear.app/myteam/issue/TEAM-123/queue-marks-items-as-failed`
- Full URL without slug: `https://linear.app/myteam/issue/TEAM-123`

The input can come from skill arguments or from the user's message directly.

Examples:
- `/linear-branch TEAM-123`
- `/linear-branch https://linear.app/myteam/issue/TEAM-123/queue-marks-items-as-failed`
- "create a branch for TEAM-123"

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

### Step 7: Create Branch

Execute these commands in sequence:

```bash
# Fetch latest from remote
git fetch <remote>

# Switch to default branch and pull latest
git checkout <default-branch> && git pull <remote> <default-branch>

# Create and switch to new branch
git checkout -b <branch-name>
```

### Step 8: Confirm

After successful branch creation:

```
Created branch `TEAM-123/queue-marks-items-as-failed` from latest <default-branch>.

Ready to work on: [Issue Title]
```

### Step 9: Set Ticket to In Progress

After successfully creating the branch, update the Linear issue status to "In Progress":

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
| Branch already exists | Offer to check out the existing branch or reset it to latest default branch |
| Default branch detection fails | Try `main` then `master` as fallbacks, otherwise report error |
