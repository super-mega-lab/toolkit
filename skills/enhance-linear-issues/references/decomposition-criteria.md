# Issue Decomposition Criteria

Subagents: Read this file when considering whether an issue should be broken into sub-issues.

## When to Decompose

Decompose when **ALL** of these are true:
1. Issue describes 2+ distinct deliverables (not just sequential steps of one task)
2. Each deliverable is independently verifiable
3. Parallel work OR incremental delivery has clear value

## When NOT to Decompose

Do NOT decompose when ANY of these are true:
- Issue is a single coherent task (even if multi-step)
- Steps are sequential and tightly coupled (step 2 can't start without step 1's output)
- Total effort is small (< 1 day of work)
- Issue is already a sub-issue (don't nest further without strong justification)
- Issue author explicitly notes it should stay as one unit

## Implementation

When decomposing:
1. Original issue becomes the **parent**
2. Create sub-issues with `parentId` pointing to the original
3. Update original description to reference children and serve as an overview
4. Each sub-issue gets its own clear scope and acceptance criteria
5. Preserve the original issue's labels and priority on children

## Examples

### DECOMPOSE: Multi-deliverable feature

**Original:** "Add label support — create label management UI, add label picker to annotations, add label filtering to views"

**Why decompose:** Three distinct deliverables (management UI, picker, filtering). Each is independently verifiable. Can be worked on in parallel or shipped incrementally.

**Sub-issues:**
1. "Add workspace label management page" — CRUD for labels
2. "Add label picker to annotation toolbar" — apply labels to annotations
3. "Add label filtering to folder/list views" — filter by label

### DON'T DECOMPOSE: Sequential, coupled steps

**Original:** "Fix advisory lock timeout — add DIRECT_DATABASE_URL env var, update prisma.config.ts to use directUrl, optionally increase lock timeout"

**Why keep together:** Steps are tightly sequential (env var must exist before config can reference it). Total effort is small. Single coherent fix for one root cause.

### DON'T DECOMPOSE: Single task with multiple aspects

**Original:** "Grouping expansions have repeated text — might need post-LLM scrubbing or prompt tweaks"

**Why keep together:** One problem, one investigation. The "scrubbing or prompt tweaks" are alternative approaches to the same fix, not separate deliverables.

### DECOMPOSE: Bug affecting multiple systems

**Original:** "Dark mode colors are wrong — sidebar uses hardcoded colors, concept tree doesn't respect theme, settings page has white backgrounds"

**Why decompose:** Three independent UI areas, each independently fixable and verifiable. Could be assigned to different people.

### DON'T DECOMPOSE: Small task

**Original:** "Update the logo on the landing page and in the sidebar"

**Why keep together:** Two instances of the same trivial change. Less than an hour of work. Creating sub-issues would be overhead for no benefit.

## Recommendation Confidence

When proposing decomposition, state confidence:
- **High confidence:** Clear multiple deliverables, obvious parallel value
- **Medium confidence:** Could go either way — present reasoning, let user decide
- **Low confidence:** Lean toward keeping together — mention as possibility but recommend against
