# Super Mega Lab Toolkit

General-purpose developer toolkit for AI coding agents. Works with Claude Code, Cursor, Windsurf, Codex, Cline, AMP, Copilot, and 18+ other agents.

## Install

### skills.sh (works with all agents)

```bash
npx skills add super-mega-lab/toolkit
```

### Claude Code plugin

```bash
/plugin marketplace add super-mega-lab/toolkit-marketplace
/plugin install sml-toolkit@sml-marketplace
```

Or install directly:

```bash
/plugin marketplace add super-mega-lab/toolkit
/plugin install sml-toolkit@sml-toolkit-dev
```

## Updating

Claude Code never updates third-party plugins silently — a new version only reaches you after the marketplace version is bumped (see [Releasing](#releasing)). Once one is published:

**Manually:**

```bash
/plugin marketplace update sml-marketplace
/plugin update sml-toolkit@sml-marketplace
```

**Automatically (per machine):** run `/plugin`, open the **Marketplaces** tab, select `sml-marketplace`, and enable auto-update. Claude Code then checks for new versions at startup and prompts you to run `/reload-plugins` when one installs. (Auto-update is off by default for non-Anthropic marketplaces.)

Replace `sml-marketplace` with `sml-toolkit-dev` if you installed via the direct path above.

## What's inside

| Type | Path | Works with |
|------|------|------------|
| Skills | `skills/` | All agents via skills.sh |
| Agents | `agents/` | Claude Code |
| Commands | `commands/` | Claude Code |
| Hooks | `hooks/` | Claude Code |

## Skills

| Skill | Description |
|-------|-------------|
| [enhance-linear-issues](skills/enhance-linear-issues/) | Review, enhance, and decompose Linear issues with clearer writing, better structure, and actionable detail |
| [linear-branch](skills/linear-branch/) | Create a git feature branch from a Linear issue, using a consistent naming convention and setting the issue to In Progress |
| [linear-worktree](skills/linear-worktree/) | Create an isolated git worktree from a Linear issue, with auto project setup, baseline tests, and the issue set to In Progress |

## Releasing

The version string is the signal Claude Code uses to detect an update — if it doesn't change, clients keep the cached copy no matter how many commits land. Three fields must stay in sync at the **same** value:

| File | Repo | Field |
|------|------|-------|
| `.claude-plugin/plugin.json` | `toolkit` | `version` |
| `.claude-plugin/marketplace.json` | `toolkit` | `plugins[0].version` (`sml-toolkit-dev`) |
| `.claude-plugin/marketplace.json` | `toolkit-marketplace` | `plugins[0].version` (`sml-marketplace`) |

To cut a release:

1. Bump all three `version` fields to the same new value.
2. Commit and push **`toolkit`** — this publishes the new plugin content (the marketplace's `url` source tracks `toolkit`'s default branch, so the code must be live there first).
3. Commit and push **`toolkit-marketplace`** — the bumped `version` here is what users' clients read to detect the new release.

Keep all three equal rather than relying on which field takes precedence — that way the detected version is unambiguous.

## License

MIT
