# Ash Kindle Defaults

These are the default values used when the user accepts defaults without customization.

## usage_rules Configuration

For **umbrella** projects, this goes in the root `mix.exs`:

```elixir
defp usage_rules do
  [
    file: "CLAUDE.md",
    usage_rules: [
      :elixir,
      :otp,
      :phoenix
    ],
    skills: [
      location: ".claude/skills",
      build: [
        "ash-framework": [
          description: "Use when making any domain, resource, or Ash-related changes.",
          usage_rules: [:ash, ~r/^ash_/]
        ]
      ]
    ]
  ]
end
```

For **single app** projects, the same config goes in the app's `mix.exs`.

## Dependencies

| Package | Version | Where | Env |
|---------|---------|-------|-----|
| `usage_rules` | `~> 1.2` | root (umbrella) or app | `:dev` only, `runtime: false` |
| `tidewave` | `~> 0.5` | web app (umbrella) or app | `:dev` only |
| `ash_ai` | `~> 0.5` | core app (umbrella) or app | all envs |

## Tidewave

- Port: whatever Phoenix is configured to use (default `4000`)
- MCP URL: `http://localhost:<port>/tidewave/mcp`

## CLAUDE.md Sections

The following sections are always included:

1. **Ash First**
2. **Code Generation**
3. **Feedback Loop** (with Tidewave + Ash AI verification steps)
4. **Logging** (wide events pattern)

For umbrella projects, also include:

5. **Umbrella Structure**

## Skills

### Always generated:

- `ash-framework` — bundles `:ash` and all `ash_*` dependency usage rules

### Suggested when deps are detected:

| If dep exists | Suggest skill | Bundles |
|---------------|--------------|---------|
| `oban` | domain-specific skill | `:oban` + related deps |
| `req` | domain-specific skill | `:req` + related deps |
| `langchain` | domain-specific skill | `:langchain` + related deps |

The skill name and description are determined by asking the user what domain
the deps serve (e.g., "scraper-pipeline", "api-client", etc.).

### Always installed externally:

```bash
npx @anthropic-ai/claude-code skills add https://github.com/boristane/agent-skills --skill logging-best-practices
```

## direnv

Skipped by default. When opted in:

```bash
# .envrc
use flake
dotenv_if_exists
```
