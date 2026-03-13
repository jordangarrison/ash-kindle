# Ash Kindle Setup Guide

Detailed reference for each setup step. The skill reads this for implementation details.

## Prerequisites

- An Ash/Phoenix project (new or existing)
- Claude Code installed
- Hex packages available (`mix deps.get` works)
- Node.js/npx available (for installing external skills)

## 1. Dependency Installation

### Umbrella Projects

Umbrella projects have multiple `mix.exs` files. Dependencies go in specific places:

**Root `mix.exs`:**
```elixir
defp deps do
  [
    # ... existing deps
    {:usage_rules, "~> 1.2", only: :dev, runtime: false}
  ]
end
```

Also add to the `project/0` function:
```elixir
def project do
  [
    # ... existing config
    listeners: [Phoenix.CodeReloader],
    usage_rules: usage_rules()
  ]
end
```

**Web app `mix.exs`** (e.g., `apps/my_app_web/mix.exs`):
```elixir
defp deps do
  [
    # ... existing deps
    {:tidewave, "~> 0.5", only: :dev}
  ]
end
```

**Core app `mix.exs`** (e.g., `apps/my_app/mix.exs`):
```elixir
defp deps do
  [
    # ... existing deps
    {:ash_ai, "~> 0.5"}
  ]
end
```

### Single App Projects

All three deps go in the single `mix.exs`:
```elixir
defp deps do
  [
    # ... existing deps
    {:usage_rules, "~> 1.2", only: :dev, runtime: false},
    {:tidewave, "~> 0.5", only: :dev},
    {:ash_ai, "~> 0.5"}
  ]
end
```

And add to `project/0`:
```elixir
def project do
  [
    # ... existing config
    listeners: [Phoenix.CodeReloader],
    usage_rules: usage_rules()
  ]
end
```

## 2. usage_rules Configuration

The `usage_rules/0` function defines what gets generated:

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

### Adding Additional Skills

When domain-specific deps are detected, suggest grouping them into skills:

```elixir
skills: [
  location: ".claude/skills",
  build: [
    "ash-framework": [
      description: "Use when making any domain, resource, or Ash-related changes.",
      usage_rules: [:ash, ~r/^ash_/]
    ],
    "background-jobs": [
      description: "Use when working on background jobs and task queues.",
      usage_rules: [:oban]
    ]
  ]
]
```

## 3. Tidewave Endpoint Setup

In the web app's `endpoint.ex`, add the Tidewave plug. It must go **before** the
router plug but **after** static file serving:

```elixir
defmodule MyAppWeb.Endpoint do
  use Phoenix.Endpoint, otp_app: :my_app_web

  # ... existing plugs (session, static, etc.)

  # Tidewave MCP server (dev only)
  if Code.ensure_loaded?(Tidewave) do
    plug Tidewave
  end

  plug MyAppWeb.Router
end
```

The conditional `Code.ensure_loaded?/1` check ensures Tidewave is only loaded in dev
where the dependency is available.

## 4. MCP Configuration

Create `.mcp.json` in the project root:

```json
{
  "mcpServers": {
    "tidewave": {
      "type": "http",
      "url": "http://localhost:4000/tidewave/mcp"
    }
  }
}
```

Adjust the port if the Phoenix app uses a non-standard port.

Claude Code reads this file to discover MCP servers. When the Phoenix server is
running in dev, Tidewave provides these tools:

- `project_eval` — evaluate Elixir expressions in the running app
- `get_ecto_schemas` — list all Ecto schemas
- `get_ash_resources` — list all Ash resources
- `execute_sql_query` — run SQL queries against the dev database
- `get_logs` — retrieve application logs
- `get_source_location` — find source file locations for modules
- `search_package_docs` — search hex package documentation
- `get_docs` — get docs for specific modules/functions

## 5. CLAUDE.md Template

The CLAUDE.md file has two parts:
1. **Hand-written sections** (project-specific guidance)
2. **Generated sections** (appended by `mix usage_rules.gen`)

### Hand-written sections for all projects:

```markdown
## Ash First

Always use Ash concepts. Never use Ecto directly. Think hard about the
"Ash way" to do things. If unsure, consult the ash-framework skill or
search docs with `mix usage_rules.search_docs`.

## Code Generation

Use `mix igniter.install` and Ash generators as a starting point wherever
possible. Modify generated code rather than writing from scratch.

## Feedback Loop

After every change:
1. Run `mix compile` — fix all errors before proceeding
2. Run `mix format --check` — fix formatting
3. Run `mix credo --strict` — address all warnings
4. Run `mix test` for affected modules
5. Use Tidewave `project_eval` to verify runtime behaviour
6. Use Ash AI MCP to verify resource/domain state

If checks fail, self-correct and retry. After 5 failed attempts on the
same issue, stop and summarise what was tried, then hand back to the human.

## Logging

Follow the wide events (canonical log lines) pattern. Emit one structured
log event per unit of work at completion. Never scatter log lines
throughout a function.
```

### Additional section for umbrella projects:

```markdown
## Umbrella Structure

- Business logic and Ash resources live in `<core_app>` only
- `<web_app>` and other apps call `<core_app>` — never the reverse
- Never write direct Ecto queries outside `<core_app>`
```

Replace `<core_app>` and `<web_app>` with actual app names.

## 6. Generation

After all configuration is in place:

```bash
mix deps.get
mix usage_rules.gen
```

This generates:
- Usage rules content appended to CLAUDE.md (between `<!-- usage-rules-start -->` and `<!-- usage-rules-end -->` markers)
- Skill files in `.claude/skills/` based on the `build` config

## 7. External Skills

Install the logging best practices skill:

```bash
npx skills add https://github.com/boristane/agent-skills --skill logging-best-practices
```

This provides Claude with guidance on the wide events / canonical log lines pattern
referenced in the CLAUDE.md Logging section.

## 8. direnv / Nix (Optional)

For Nix users, create `.envrc`:

```bash
use flake
dotenv_if_exists
```

This assumes:
- A `flake.nix` exists with the Elixir/Erlang/Node toolchain
- `direnv` is installed and hooked into the shell
- The user may have a `.env` file for local secrets

This step is skipped by default since Nix setup varies significantly per environment.

## 9. Verification Checklist

After setup, confirm:

- [ ] `mix deps.get` succeeds
- [ ] `mix compile` succeeds with no warnings
- [ ] `mix format --check-formatted` passes
- [ ] `CLAUDE.md` exists with hand-written + generated sections
- [ ] `.claude/skills/ash-framework/` directory exists with generated skill
- [ ] `.mcp.json` exists with Tidewave config
- [ ] `logging-best-practices` skill is installed
- [ ] (If opted in) `.envrc` exists
- [ ] Starting the Phoenix server (`mix phx.server`) makes Tidewave available at the configured URL
