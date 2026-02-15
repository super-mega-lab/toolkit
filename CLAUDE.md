# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Is

A Claude Code plugin (`sml-toolkit`) that provides skills, agents, commands, and hooks for AI coding agents. Published via both the Claude Code plugin system and skills.sh. Currently at v0.1.0.

## Repository Structure

```
.claude-plugin/     # Plugin manifest (plugin.json) and dev marketplace config (marketplace.json)
skills/             # Skill definitions — each skill is a directory with a SKILL.md
agents/             # Agent definitions (empty — .gitkeep)
commands/           # Command definitions (empty — .gitkeep)
hooks/              # Hook definitions (empty — .gitkeep)
```

## Architecture

**Skills** are the primary content type. Each skill lives in `skills/<skill-name>/` and is defined by a `SKILL.md` file with YAML frontmatter (`name`, `description`) followed by the full prompt/instructions the agent receives when the skill is invoked.

**Plugin config** in `.claude-plugin/plugin.json` defines the package metadata. Version must be kept in sync across `plugin.json` and `marketplace.json`.

## Current Skills

- **enhance-linear-issues**: Reviews and enhances Linear issues via the Linear MCP server. Uses single-agent sequential processing for ≤10 issues, with parallel subagent dispatch as fallback for larger batches. Calibration scales and decomposition criteria are inlined in the SKILL.md.

## Development Notes

- No build step, no test framework, no linter — this is a content-driven plugin (markdown + JSON).
- Changes to skill behavior are made by editing SKILL.md files.
- The `enhance-linear-issues` skill has a version number in its SKILL.md that controls idempotency (the "already processed" filter). Bump it when making behavioral changes.
