---
name: enhance-linear-issues
description: Use when asked to review, enhance, improve, clean up, decompose, or organize Linear issues. Handles requests mentioning "tickets" or "tasks" — these mean Linear issues. Triggers on "enhance issues", "enhance tickets", "improve issues", "improve tickets", "clean up issue TEAM-123", "break down this issue", or requests to improve issue quality, structure, or organization.
---

# Enhance Linear Issues

**Version: 1** — This is the single source of truth for the skill version. All version references below mean this value.

## Overview

Enhance hastily-written Linear issues with clearer writing, better structure, and actionable detail — without over-engineering or losing the author's voice. Optionally decompose complex issues into sub-issues and link related issues.

**Core principle:** Enhance to the issue's potential, not to some idealized template. A quick feature idea shouldn't become an implementation spec.

**Workspace limitation:** Linear MCP tools operate on the authenticated user's workspace implicitly. There is no workspace switching — "team-aware" means discovering teams/projects in the current workspace, not switching between workspaces.

## When to Use

Use this skill when the user mentions Linear issues **and** wants them reviewed, enhanced, improved, cleaned up, decomposed, or organized.

**Trigger phrases:** "enhance issues", "review my issues", "clean up", "improve issue quality", "break down this issue", "decompose", "organize issues"

**NOT for:** Creating new issues from scratch, triaging, or status updates.

## Workflow

```
Enhancement Progress:
- [ ] Step 1: Prerequisites checked
- [ ] Step 2: Issues gathered, workspace context loaded
- [ ] Step 3: Scope confirmed with user
- [ ] Step 4: Already-processed issues filtered
- [ ] Step 5: Sub-tasks dispatched
- [ ] Step 6: Proposals collected and presented
- [ ] Step 7: Safety review complete
- [ ] Step 8: Approval obtained
- [ ] Step 9: Changes applied
```

### Phase 1 — Discover and Analyze

#### Step 1: Announce and Select Mode

1. Display: `🎫 enhance-linear-issues v[current version]`
2. **Check prerequisites** (run in parallel):
   - **Linear MCP**: Call `Linear:list_teams`. If the tool is unavailable or the call fails, stop with: "Linear MCP server is required but not available. Ensure the Linear MCP server is configured in your agent's MCP settings."
   - **Git**: Run `git --version` in the shell. If it fails, note that file linking will be unavailable (the skill can still enhance issues without repo links). Include this in the scope summary.
3. Detect mode from user's request:
   - **Auto** (no trigger needed): Runs end-to-end without prompts. Auto-applies all proposals that pass safety review. Flagged proposals are skipped (not applied) and reported.
   - **Preview** (opt-in): Runs the full analysis pipeline without prompts, then presents results for review before applying.

Preview triggers: "preview", "show me first", "don't apply yet", "dry run"

If mode is ambiguous, default to auto.

#### Step 2: Gather Issues and Workspace Context

**Run in parallel:**

**A. Gather issues** based on user's criteria:
- "issues I created today" → `Linear:list_issues` with `createdAt` filter and `assignee: "me"`
- "issues in backlog" → `Linear:list_issues` with `state: "backlog"`
- "issues in project X" → `Linear:list_issues` with `project` filter
- Specific issue → `Linear:get_issue` with identifier

If `Linear:list_issues` returns paginated results, fetch all pages to build the complete issue set before proceeding.

**B. Discover workspace context:**
- `Linear:list_teams` — fetch all teams (for team reassignment recommendations)
- `Linear:list_projects` — fetch all projects (for context and linking)

**C. Discover git/repo context** (run in the shell):
- `git remote get-url origin` → remote URL
- `git rev-parse --abbrev-ref HEAD` → current branch (fallback: `main`)
- Parse remote URL to browsable base:
  - SSH format `git@github.com:org/repo.git` → `https://github.com/org/repo`
  - HTTPS format `https://github.com/org/repo.git` → `https://github.com/org/repo`
- Construct `REPO_BROWSE_URL` = `{base_url}/blob/{branch}`
- If git commands fail, set `REPO_BROWSE_URL` to empty and note that file linking is unavailable

**D. Discover workspace slug:**
- From the first issue fetched in A, extract the `url` field from the `Linear:get_issue` response
- Parse workspace slug from `https://linear.app/{WORKSPACE_SLUG}/issue/...`
- Fallback: extract from any Linear URL the user provided
- If unavailable, issue references will use plain text identifiers

Store all context (workspace, repo, workspace slug) for passing to sub-tasks.

#### Step 3: Confirm Scope

Show the user what will be processed:

```
Mode: [Auto/Preview]

Enhancing the following issues:
- TEAM-101: [title] (status: [status], team: [team])
- TEAM-102: [title] (status: [status], team: [team])
- TEAM-103: [title] (status: [status], team: [team])

Available teams: [team1, team2, ...]
Available projects: [project1, project2, ...]
Repo: [REPO_BROWSE_URL or "unavailable"]
Workspace: [WORKSPACE_SLUG or "unavailable"]
```

Display scope summary and proceed to analysis.

**Batching (>10 issues):**
- **Auto mode:** Process in batches of 10 automatically. After each batch, report results and continue with the next batch. No prompting.
- **Preview mode:** Prompt with options: "Process all [N] issues" or "Process in batches of 10."

#### Step 4: Filter Already-Processed Issues

Before dispatching sub-tasks, check each issue's description for `_Last enhanced: v[current version],`:
- If the version matches **AND** the issue's `updatedAt` timestamp is within ~1 minute of the footer timestamp → **skip** this issue
- Report: "Already processed by current version, no changes since last run"
- Otherwise → proceed to sub-task dispatch

This avoids redundant processing when re-running the skill on the same batch.

#### Step 5: Dispatch Sub-Tasks for Analysis

**CRITICAL:** Dispatch a separate parallel sub-task for each issue. This prevents context degradation when processing many issues.

Each sub-task receives:
- The issue ID
- Workspace context (teams, projects)
- All other issue IDs in the batch (for cross-linking)
- Instructions to follow the Sub-Task Enhancement Process below

Launch sub-tasks in parallel where possible.

### Sub-Task Enhancement Process

**Sub-task instructions template:**

```
Enhance Linear issue [ISSUE-ID].

## Your task
1. Fetch the issue with Linear:get_issue (include relations)
2. Fetch comments with Linear:list_comments
3. Read the enhancement calibration reference at [skill base dir]/references/enhancement-calibration.md
4. Assess issue quality using the calibration scale
5. If quality is Excellent, add enhancement footer and report, but make no content changes
6. Identify issue type (bug/feature/task)
7. Gather context:
   - If issue has images in description, use Linear:extract_images to view them, then describe what they show
   - If screenshot analysis fails, note "Screenshot attached (not analyzed)" — do NOT guess content
   - Search for related issues among: [list of other issue IDs in batch]
   - Search for related issues via Linear:list_issues with similar terms
   - If issue mentions a feature/bug area, check project documentation and relevant source files
   - Verify all code references before citing them
8. Read decomposition criteria at [skill base dir]/references/decomposition-criteria.md
9. Evaluate whether this issue should be decomposed into sub-issues
10. Check if the issue's team assignment makes sense given available teams: [team list]
11. Draft enhanced title and description (calibrated to quality gap)
12. Run safety checks (see Safety Review section)
13. Return results in the output format specified below

## Linking Rules

Use these URL patterns for all references in descriptions:
- **Issues**: `https://linear.app/WORKSPACE_SLUG/issue/ISSUE-ID` — WORKSPACE_SLUG: [dynamic]
- **Source files**: `REPO_BROWSE_URL/file#Lstart-Lend` — REPO_BROWSE_URL: [dynamic]
- **Documents**: `REPO_BROWSE_URL/path/to/document`
- If slugs/URLs are unavailable, fall back to plain text

## Enhancement Footer

ALWAYS append a single footer line to the description, even for NoChangesNeeded issues. Format:

    _Last enhanced: v[VERSION], [ISO-8601-TIMESTAMP]_

Example: `_Last enhanced: v1, 2024-01-15T12:00:00Z_`

Rules:
- If description already has a `_Last enhanced:` line, replace it (never duplicate)
- Remove any old-format blocks (`<enhance-linear-issues>`, `<EnhanceLinearSkills>`) during enhancement
- Use the current version number and UTC timestamp

## Available teams
[dynamic team list]

## Available projects
[dynamic project list]

## Other issues in this batch (for cross-linking)
[dynamic issue list with titles]

## Guard Rails — follow these strictly
[Include full Guard Rails section]
```

### Phase 2 — Review and Apply

#### Step 6: Collect and Present Proposals

After all sub-tasks complete, collect their proposals and present a **unified change plan** showing all issues at once:

```markdown
# Enhancement Plan

**Mode:** [Auto/Preview]
**Issues analyzed:** [N]
**Proposed changes:** [N] (Skipped: [N], Flagged: [N])

[For each issue, use the Output Format below]
```

#### Step 7: Safety Review

Run automated sanity checks on each proposal before presenting/applying:

| Check | Fail Condition | Action |
|-------|---------------|--------|
| Description shrinkage | Proposed description >20% shorter than original | FLAG |
| Title radical change | Edit distance >50% of original title length | FLAG |
| Data loss | Screenshots/links in original but missing in proposal | FLAG |
| Invalid team | Team reassignment to non-existent team | FLAG |
| Orphaned sub-issues | Decomposition creates children without updating parent description | FLAG |
| Circular relations | `relatedTo` would create self-reference | FLAG |

**Auto mode:** Flagged proposals are skipped (not applied) and reported in the summary.
**Preview mode:** Flagged proposals require explicit selection even when choosing "Apply all passing."

#### Step 8: Approval

**Auto mode:**
1. Auto-apply all proposals that pass safety review (PASS status)
2. Skip any flagged proposals — do not apply them
3. Report what was applied and what was skipped (with reasons)
4. No user interaction at any point

**Preview mode:**
1. Present the unified change plan with safety status per issue
2. Show a single menu prompt with these options:
   - **Complete preview** (default) — Report only, no changes applied
   - **Apply all passing** — Apply all proposals that passed safety review
   - **Choose which to apply** — Let user pick specific issues by ID
3. Flagged proposals require explicit selection even with "Apply all passing"

#### Step 9: Apply Changes

For each approved issue:
1. Call `Linear:update_issue` with new title and/or description
2. If decomposing: create sub-issues with `Linear:create_issue` using `parentId`, then update parent description
3. If linking related issues: call `Linear:update_issue` with `relatedTo`
4. Confirm what was changed

**Error recovery:** If `Linear:update_issue` fails for an issue, log the error, continue with remaining issues, and report failures in the summary. Never stop the entire batch for a single failure.

## Guard Rails

### DO NOT:

1. **Remove anything** — Screenshots, comments, links, attachments stay intact
2. **Invent details** — If you don't know, don't guess. Say "needs investigation"
3. **Over-specify features** — Don't turn "add labels" into a Prisma schema
4. **Sanitize voice** — "I clicked like a crazy person" is personality, not a grammar error
5. **Add implementation details** unless the issue is clearly meant to be a spec
6. **Assume vagueness is error** — Sometimes flexibility is intentional
7. **Auto-assign to individuals** — We lack workload knowledge; only recommend team reassignment
8. **Nest sub-issues** — Don't decompose issues that are already sub-issues without strong justification

### DO:

1. **Fix actual errors** — Typos, grammar, trailing punctuation
2. **Add structure** — Headers make long descriptions scannable
3. **Describe screenshots** — "Screenshot shows: repeated text in grouping expansion for segment S-14"
4. **Link related issues** — If you find related issues, add them via `relatedTo`
5. **Add enhancement footer** — Always append `_Last enhanced: v[VERSION], [TIMESTAMP]_`; replace existing; never duplicate
6. **Cite specific code as links** — Use REPO_BROWSE_URL for file:line references, but ONLY if verified
7. **Recommend team changes** — When an issue clearly belongs on a different team, say so with reasoning

## Output Format

For each issue, report under a `## TEAM-XXX: [Title]` heading:

- **Quality assessment** and **recommendation** (NoChangesNeeded/LightTouch/Moderate/Significant/Decomposed)
- **Title**: current vs proposed (or NO CHANGE)
- **Description**: current vs proposed (or NO CHANGE)
- **Sub-issues** (if decomposing): child titles, descriptions, and parent update
- **Related issues**: issue IDs with relationship reason
- **Team assignment**: current vs proposed with reasoning (or NO CHANGE)
- **Safety check**: PASS or FLAG with explanation
- **Changes summary**: what changed and why

## Calibration and Decomposition References

Enhancement calibration (quality scales, issue-type guidance, common mistakes, examples):
→ Read `references/enhancement-calibration.md`

Decomposition criteria (when to decompose, when not to, examples):
→ Read `references/decomposition-criteria.md`
