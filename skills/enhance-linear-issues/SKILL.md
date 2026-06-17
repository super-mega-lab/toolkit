---
name: enhance-linear-issues
description: Use when asked to review, enhance, improve, clean up, decompose, or organize Linear issues. Handles requests mentioning "tickets" or "tasks" — these mean Linear issues. Triggers on "enhance issues", "enhance tickets", "improve issues", "improve tickets", "clean up issue TEAM-123", "break down this issue", or requests to improve issue quality, structure, or organization.
---

# Enhance Linear Issues

**Version: 3** — This is the single source of truth for the skill version. All version references below mean this value.

## Overview

Enhance hastily-written Linear issues with clearer writing, better structure, and actionable detail — without over-engineering or losing the author's voice. Optionally decompose complex issues into sub-issues and link related issues.

**Core principle:** Enhance to the issue's potential, not to some idealized template. A quick feature idea shouldn't become an implementation spec.

**Workspace limitation:** Linear MCP tools operate on the authenticated user's workspace implicitly. There is no workspace switching — "team-aware" means discovering teams/projects in the current workspace, not switching between workspaces.

## When to Use

**NOT for:** Creating new issues from scratch, triaging, or status updates.

## Linear MCP Tools

Load all needed tools with a single ToolSearch call: `+linear`. Tools used by this skill:

- `list_teams` — list workspace teams
- `list_projects` — list workspace projects
- `list_issues` — list/filter issues
- `get_issue` — get issue details (supports `includeRelations`)
- `list_comments` — get issue comments
- `extract_images` — extract and view images from issue descriptions
- `save_issue` — create or update issues (updates when an `id` is passed, creates when omitted; used for field updates and sub-issue creation)

## Workflow

```
Enhancement Progress:
- [ ] Step 1: Prerequisites checked
- [ ] Step 2: Issues gathered, workspace context loaded
- [ ] Step 3: Scope confirmed with user
- [ ] Step 4: Already-processed issues filtered
- [ ] Step 5: Issues processed
- [ ] Step 6: Proposals collected and presented
- [ ] Step 7: Safety review complete
- [ ] Step 8: Approval obtained
- [ ] Step 9: Changes applied
```

### Phase 1 — Discover and Analyze

#### Step 1: Announce and Select Mode

1. Display: `🎫 enhance-linear-issues v[current version]`
2. **Check git prerequisite**: Run `git --version`. If it fails, note that file linking will be unavailable. Include this in the scope summary.
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
- `Linear:list_teams` — fetch all teams (for team reassignment). If this call fails, stop with: "Linear MCP server is required but not available. Ensure the Linear MCP server is configured in your agent's MCP settings."
- `Linear:list_projects` — fetch all projects (for context and linking)

**C. Discover git/repo context** (run in the shell):
- Construct `REPO_BROWSE_URL` from `git remote get-url origin` and current branch: `{https-base}/blob/{branch}`
- If git commands fail, set `REPO_BROWSE_URL` to empty and note file linking is unavailable

**D. Discover workspace slug:**
- Extract `WORKSPACE_SLUG` from any issue's `url` field (e.g., `https://linear.app/{WORKSPACE_SLUG}/issue/...`). If unavailable, fall back to plain text identifiers.

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

Before processing, check each issue's description for an enhancement footer. Linear normalizes underscores to asterisks on save, so the stored footer may be `_Last enhanced: ..._` **or** `*Last enhanced: ...*` — match **both** forms. The footer format is `Last enhanced: v[version], [timestamp], [hash]` (see Enhancement Footer).

For each issue:
1. **No footer, or footer version ≠ current version** → proceed to Step 5 (process normally).
2. **Footer present with the current version** → recompute the content hash and compare:
   - Strip the footer line (and any trailing blank lines) from the description.
   - Hash the remaining description with the exact method in the Enhancement Footer section (SHA-256, first 12 hex chars, computed in the shell — never estimate the hash).
   - **Hashes equal** → **skip** this issue. Report: "Already processed by v[version], no content changes since last run."
   - **Hashes differ** → the description changed since the last enhancement → proceed to Step 5.

Do **NOT** use `updatedAt` for dedup: creating a comment bumps `issue.updatedAt` even when the description is unchanged, so a timestamp check would needlessly re-enhance. The content hash is immune to markdown normalization, clock skew, and comment-driven `updatedAt` bumps.

This avoids redundant processing when re-running the skill on the same batch.

#### Step 5: Process Issues

Choose a processing strategy based on batch size:

**Small batches (≤10 issues):** Process each issue sequentially in the current agent using the Enhancement Process below. Context (tools, calibration criteria, workspace info) loads once and is reused across all issues.

**Large batches (>10 issues):** Dispatch a separate parallel sub-task for each issue to prevent context degradation. Each sub-task prompt must include:
- The issue ID and the Enhancement Process steps below
- The full Enhancement Calibration and Decomposition Criteria sections from this skill
- The Guard Rails, Linking Rules, and Enhancement Footer sections
- Dynamic context: workspace slug, repo browse URL, team list, project list, batch issue list

Launch sub-tasks in parallel where possible.

### Enhancement Process

For each issue (whether processed inline or via sub-task):

1. Fetch the issue with `Linear:get_issue` (include relations) — reuse data from Step 2 when available
2. Fetch comments with `Linear:list_comments`
3. Assess issue quality using the Enhancement Calibration scale (see below)
4. If quality is Excellent (NoChangesNeeded), make no content changes and do **NOT** write the issue — report it as skipped. Never write an issue whose content is unchanged (see Enhancement Footer)
5. Identify issue type (bug/feature/task)
6. Gather context:
   - If issue has images in description, use `Linear:extract_images` to view them, then describe what they show
   - If screenshot analysis fails, note "Screenshot attached (not analyzed)" — do NOT guess content
   - Search for related issues among other issue IDs in the batch
   - Search for related issues via `Linear:list_issues` with similar terms
   - If issue mentions a feature/bug area, check project documentation and relevant source files
   - Verify all code references before citing them
7. Evaluate whether this issue should be decomposed (see Decomposition Criteria below)
8. Check if the issue's team assignment makes sense given available teams
9. Draft enhanced title and description (calibrated to quality gap)
10. Run safety checks (see Safety Review section)
11. Return results in the Output Format specified below

#### Linking Rules

Use these URL patterns for all references in descriptions:
- **Issues**: `https://linear.app/WORKSPACE_SLUG/issue/ISSUE-ID` — WORKSPACE_SLUG: [dynamic]
- **Source files**: `REPO_BROWSE_URL/file#Lstart-Lend` — REPO_BROWSE_URL: [dynamic]
- **Documents**: `REPO_BROWSE_URL/path/to/document`
- If slugs/URLs are unavailable, fall back to plain text

#### Enhancement Footer

**Only changed issues get a footer. Never write an issue whose content is unchanged** — no footer-only writes, no pristine writes. If enhancement produces zero content changes (e.g., a NoChangesNeeded issue), skip the `Linear:save_issue` call entirely and report it as skipped. Re-sending a full description just to stamp a footer is wasteful and risks corrupting `<issue>` mention tags on the round-trip.

For issues that DO change, the footer carries a content hash so re-runs can reliably detect "already enhanced, nothing changed." Format:

    _Last enhanced: v[VERSION], [ISO-8601-TIMESTAMP], [HASH]_

Example: `_Last enhanced: v3, 2024-01-15T12:00:00Z, a1b2c3d4e5f6_`

`[HASH]` is the first 12 hex characters of the SHA-256 of the **stored** description with the footer line removed. Because Linear rewrites markdown on save (e.g., `_x_` → `*x*`), the hash MUST be computed over the post-save form. This requires a save-then-rehash sequence for each changed issue:

1. Save the enhanced description **without** a footer: `Linear:save_issue(id, description=<enhanced body>)`.
2. Re-fetch with `Linear:get_issue` to read the normalized stored description.
3. Hash that normalized description (it has no footer yet) deterministically in the shell — never estimate it. For example, write it to a temp file and run `sha256sum <file> | cut -c1-12`.
4. Append the footer and save once more: `Linear:save_issue(id, description=<normalized body> + "\n\n" + "_Last enhanced: v[VERSION], [UTC-TIMESTAMP], [HASH]_")`.

On a later run, Step 4 strips this footer, hashes the remaining (already-normalized) body, and gets the same hash — so an unchanged issue is skipped.

Rules:
- Match **both** `_Last enhanced: ..._` and `*Last enhanced: ...*` when detecting or replacing an existing footer (never duplicate).
- When stripping the footer to compute or compare a hash, remove the footer line and any trailing blank lines so the hashed body is stable across runs.
- Remove any old-format blocks (`<enhance-linear-issues>`, `<EnhanceLinearSkills>`) and any legacy v2 footer (`_Last enhanced: v2, [timestamp]_`, no hash) during enhancement.
- Use the current version number and UTC timestamp.

### Phase 2 — Review and Apply

#### Step 6: Collect and Present Proposals

After all issues are processed, collect the proposals and present a **unified change plan** showing all issues at once:

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
1. **Skip unchanged issues** — if an issue has no content changes (title and description identical to current), do NOT call `Linear:save_issue`. Report it as skipped.
2. For issues with content changes, apply the new title and/or description, then stamp the content-hash footer using the save-then-rehash sequence in the Enhancement Footer section.
3. If decomposing: create sub-issues with `Linear:save_issue` (no `id`, with `parentId`), then update the parent description with `Linear:save_issue` (with `id`)
4. If linking related issues: call `Linear:save_issue` with the issue `id` and `relatedTo`
5. Confirm what was changed

**Error recovery:** If `Linear:save_issue` fails for an issue, log the error, continue with remaining issues, and report failures in the summary. Never stop the entire batch for a single failure.

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
5. **Add enhancement footer to changed issues** — For issues with content changes, append `_Last enhanced: v[VERSION], [TIMESTAMP], [HASH]_` (see Enhancement Footer); replace any existing footer; never duplicate. Never write a footer to an unchanged issue
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

## Enhancement Calibration

### Quality Assessment Scale

| Current Quality | Enhancement Level | What It Looks Like |
|-----------------|-------------------|--------------------|
| **Excellent** (structured, clear, actionable) | NoChangesNeeded | Has Problem/Expected/Criteria sections, code refs, links |
| **Good** (clear intent, some structure) | LightTouch: grammar, minor clarity | Fix typos, preserve author's structure |
| **Medium** (understandable but sparse) | Moderate: add structure, clarify | Add sections, expand terse points |
| **Poor** (screenshot + few sentences) | Significant: full restructure | Add Problem/Causes/Investigation sections |

### By Issue Type

**Bug Issues** benefit from:
- Problem statement (what's broken, where)
- Reproduction steps (if known or inferable)
- Expected vs actual behavior
- Investigation pointers (relevant files/functions — verified only)
- Possible causes (if codebase context helps)

**Feature Issues** benefit from:
- Clear description of desired outcome
- Reasoning/motivation (preserve if present)
- Acceptance criteria (if straightforward to infer)
- **NOT** implementation specs unless issue is clearly meant to be a spec

**Task Issues** benefit from:
- Clear definition of done
- Context links (related issues, docs)

### Common Mistakes

| Mistake | Example | Fix |
|---------|---------|-----|
| Over-enhancement | Turning feature request into implementation spec | Match enhancement to issue type |
| Lost personality | Removing casual language | Only fix actual grammar errors |
| Hallucinated details | "See line 165 in foo.ts" (wrong) | Verify every code reference |
| Scope creep | Adding "Out of Scope" sections to simple issues | Keep additions proportional |
| Template forcing | Every issue gets Problem/Expected/Criteria | Adapt structure to issue type |
| Description inflation | Adding filler text that doesn't add information | Every added sentence must carry new signal |

### Calibration Examples

**Poor issue (bug report) — Significant enhancement:**

Before: Title "Data export broken:" / Description: [screenshot] "Noticed this happening a few times. Maybe we should add validation?"

Appropriate: Fix trailing colon in title. Add Problem/Possible Causes/Investigation structure. Add verified file paths. Describe screenshot. Add footer.

**Good issue (feature request) — LightTouch:**

Before: Well-reasoned feature request with screenshots and flexibility notes like "feel free to simplify if needed"

Appropriate: Fix typos. Light restructure with headers IF it helps readability. Preserve flexibility language. Do NOT add implementation details. Add footer.

**Excellent issue — NoChangesNeeded:**

Before: Structured with Problem, Root Cause, Solution, Tasks, References sections. Includes exact error codes, code snippets, and documentation links.

Appropriate: Make no changes and do NOT write the issue (no footer). Report why: "Already excellent — structured with problem/cause/solution, includes code examples and references. Skipped (no content changes)."

## Decomposition Criteria

### When to Decompose

Decompose when **ALL** of these are true:
1. Issue describes 2+ distinct deliverables (not just sequential steps of one task)
2. Each deliverable is independently verifiable
3. Parallel work OR incremental delivery has clear value

### When NOT to Decompose

Do NOT decompose when ANY of these are true:
- Issue is a single coherent task (even if multi-step)
- Steps are sequential and tightly coupled (step 2 can't start without step 1's output)
- Total effort is small (< 1 day of work)
- Issue is already a sub-issue (don't nest further without strong justification)
- Issue author explicitly notes it should stay as one unit

### Decomposition Implementation

When decomposing:
1. Original issue becomes the **parent**
2. Create sub-issues with `parentId` pointing to the original
3. Update original description to reference children and serve as an overview
4. Each sub-issue gets its own clear scope and acceptance criteria
5. Preserve the original issue's labels and priority on children

### Decomposition Examples

**DECOMPOSE — Multi-deliverable feature:**
"Add label support — create label management UI, add label picker to annotations, add label filtering to views"
→ Three distinct deliverables, each independently verifiable, can be worked in parallel.

**DON'T DECOMPOSE — Sequential, coupled steps:**
"Fix advisory lock timeout — add DIRECT_DATABASE_URL env var, update prisma.config.ts to use directUrl, optionally increase lock timeout"
→ Steps are tightly sequential. Single coherent fix for one root cause.

**DON'T DECOMPOSE — Single task with multiple aspects:**
"Grouping expansions have repeated text — might need post-LLM scrubbing or prompt tweaks"
→ One problem, one investigation. Alternative approaches to the same fix.

**DECOMPOSE — Bug affecting multiple systems:**
"Dark mode colors are wrong — sidebar uses hardcoded colors, concept tree doesn't respect theme, settings page has white backgrounds"
→ Three independent UI areas, each independently fixable and verifiable.

**DON'T DECOMPOSE — Small task:**
"Update the logo on the landing page and in the sidebar"
→ Two instances of the same trivial change. Less than an hour of work.

### Recommendation Confidence

When proposing decomposition, state confidence:
- **High confidence:** Clear multiple deliverables, obvious parallel value
- **Medium confidence:** Could go either way — present reasoning, let user decide
- **Low confidence:** Lean toward keeping together — mention as possibility but recommend against
