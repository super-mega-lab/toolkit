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

## License

MIT
