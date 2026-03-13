# ash-kindle

A Claude Code skill that sets up AI-assisted development tooling for [Ash](https://ash-hq.org/) / [Phoenix](https://www.phoenixframework.org/) projects.

## What It Sets Up

| Tool | Purpose |
|------|---------|
| [usage_rules](https://hex.pm/packages/usage_rules) | Generates CLAUDE.md content and Claude Code skills from dependency usage rules |
| [Tidewave](https://hex.pm/packages/tidewave) | Dev-only MCP server for runtime introspection (eval, logs, schemas, SQL) |
| [Ash AI](https://hex.pm/packages/ash_ai) | Usage rules and MCP tooling for Ash resources |
| CLAUDE.md | Project instructions — Ash First, Feedback Loop, Logging conventions |
| Claude Code skills | Auto-generated from usage_rules config, tailored to your deps |
| [logging-best-practices](https://github.com/boristane/agent-skills) | Wide events / canonical log lines pattern |
| direnv / Nix | Optional environment setup |

## Install

```bash
npx @anthropic-ai/claude-code skills add https://github.com/jordangarrison/ash-kindle --skill ash-kindle
```

## Usage

In any Ash/Phoenix project, tell Claude:

> Set up AI dev tooling for this project

Or invoke directly:

> /ash-kindle

Claude will detect your project structure (umbrella vs single app, existing domains, Phoenix port) and walk you through the setup. Accept defaults for the standard config, or customize each piece.

## Defaults

When you accept defaults, ash-kindle configures:

- **usage_rules** with `:elixir`, `:otp`, `:phoenix` rules
- **One skill** (`ash-framework`) bundling all Ash-related usage rules
- **Tidewave** on your existing Phoenix port
- **CLAUDE.md** with Ash First, Code Generation, Feedback Loop, and Logging sections
- **logging-best-practices** external skill
- **direnv** skipped (opt-in)

## Requirements

- An Ash/Phoenix project (new or existing)
- Claude Code
- Node.js/npx (for external skill installation)
- Hex packages available
