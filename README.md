# ash-kindle

Portable agent skills for AI-assisted development in
[Ash](https://ash-hq.org/) and
[Phoenix](https://www.phoenixframework.org/) projects.

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
npx skills add https://github.com/jordangarrison/ash-kindle --skill ash-kindle
```

Install the parallel Ash/Phoenix PR reviewer:

```bash
npx skills add https://github.com/jordangarrison/ash-kindle \
  --skill ash-pr-review-team
```

Install globally for every supported agent:

```bash
npx skills add jordangarrison/ash-kindle -g -a '*' \
  --skill ash-pr-review-team -y
```

## Usage

In any Ash/Phoenix project, tell Claude:

> Set up AI dev tooling for this project

Or invoke directly:

> /ash-kindle

Claude will detect your project structure (umbrella vs single app, existing domains, Phoenix port) and walk you through the setup. Accept defaults for the standard config, or customize each piece.

For a four-perspective review of an Ash/Phoenix pull request, ask:

> Run an Ash PR review team on this pull request

The `ash-pr-review-team` skill dispatches Ash, LiveView/UI, security, and
performance/SRE reviewers in parallel, then consolidates their evidence. It is
a global fallback: when a repository provides a project-local PR review skill,
that local workflow takes precedence unless you explicitly invoke
`$ash-pr-review-team`.

Pass `--skip-validation` for a static-only fast review. This skips optional
tests and documentation lookup while retaining exact diff/head checks, the full
preview, and explicit posting sign-off.

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
- An Agent Skills-compatible coding agent
- Node.js/npx (for external skill installation)
- Hex packages available
