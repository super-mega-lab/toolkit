# Enhancement Calibration Reference

Subagents: Read this file to calibrate enhancement depth per issue.

## Quality Assessment Scale

| Current Quality | Enhancement Level | What It Looks Like |
|-----------------|-------------------|--------------------|
| **Excellent** (structured, clear, actionable) | NoChangesNeeded | Has Problem/Expected/Criteria sections, code refs, links |
| **Good** (clear intent, some structure) | LightTouch: grammar, minor clarity | Fix typos, preserve author's structure |
| **Medium** (understandable but sparse) | Moderate: add structure, clarify | Add sections, expand terse points |
| **Poor** (screenshot + few sentences) | Significant: full restructure | Add Problem/Causes/Investigation sections |

## By Issue Type

### Bug Issues

Benefit from:
- Problem statement (what's broken, where)
- Reproduction steps (if known or inferable)
- Expected vs actual behavior
- Investigation pointers (relevant files/functions — verified only)
- Possible causes (if codebase context helps)

### Feature Issues

Benefit from:
- Clear description of desired outcome
- Reasoning/motivation (preserve if present)
- Acceptance criteria (if straightforward to infer)
- **NOT** implementation specs unless issue is clearly meant to be a spec

### Task Issues

Benefit from:
- Clear definition of done
- Context links (related issues, docs)

## Common Mistakes

| Mistake | Example | Fix |
|---------|---------|-----|
| Over-enhancement | Turning feature request into implementation spec | Match enhancement to issue type |
| Lost personality | Removing casual language | Only fix actual grammar errors |
| Hallucinated details | "See line 165 in foo.ts" (wrong) | Verify every code reference |
| Scope creep | Adding "Out of Scope" sections to simple issues | Keep additions proportional |
| Template forcing | Every issue gets Problem/Expected/Criteria | Adapt structure to issue type |
| Description inflation | Adding filler text that doesn't add information | Every added sentence must carry new signal |

## Example: Calibrated Enhancement

### Poor issue (bug report)

**Before:**
```
Title: "Data export broken:"
Description: [screenshot] "Noticed this happening a few times. Maybe we should add validation?"
```

**Appropriate enhancement:**
- Fix trailing colon in title
- Add Problem/Possible Causes/Investigation structure
- Add relevant file paths (verified against codebase)
- Describe what screenshot shows
- Add enhancement footer

### Good issue (feature request)

**Before:**
```
Title: "Add label support for items"
Description: [Well-reasoned feature request with screenshots and flexibility notes like "feel free to simplify if needed"]
```

**Appropriate enhancement:**
- Fix any typos
- Light restructure with headers IF it helps readability
- Preserve flexibility language the author included
- Do NOT add implementation details
- Add enhancement footer

### Excellent issue (skip)

**Before:**
```
Title: "Fix intermittent deployment failures due to advisory lock timeout"
Description: [Structured with Problem, Root Cause, Solution, Tasks, References sections. Includes exact error codes, code snippets, and documentation links.]
```

**Appropriate enhancement:**
- Add enhancement footer but make no content changes
- Note in report why no changes were needed: "Already excellent — structured with problem/cause/solution, includes code examples and references"
