# Ash Kindle Setup Guide

Detailed reference for each setup step. The skill reads this for implementation details.

## Prerequisites

- An Ash/Phoenix project (new or existing) — or an empty directory to create one in
- Claude Code installed
- Hex packages available (`mix deps.get` works)
- Node.js/npx available (for installing external skills)

## 0. Creating a New Project (if needed)

If no `mix.exs` exists in the directory, create a new Phoenix project:

```bash
# Install the Phoenix generator
mix archive.install hex phx_new --force

# Create project in current directory (yes | to auto-confirm existing dir)
yes | mix phx.new . --app my_app
```

Key notes:
- The `yes |` prefix auto-confirms the "directory already exists" prompt
- Single app is recommended to start — umbrella can be migrated to later
- Ash's domain/resource architecture provides strong separation within a single app

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

All deps go in the single `mix.exs`:
```elixir
defp deps do
  [
    # ... existing deps

    # Explicit plug to resolve tidewave/ash_json_api env conflict
    {:plug, "~> 1.19"},

    # Ash
    {:ash, "~> 3.0"},
    {:ash_phoenix, "~> 2.0"},
    {:ash_postgres, "~> 2.0"},
    {:ash_ai, "~> 0.5"},

    # AI dev tooling
    {:usage_rules, "~> 1.2", only: :dev, runtime: false},
    {:tidewave, "~> 0.5", only: :dev}
  ]
end
```

### Known Issue: `plug` Dependency Conflict

When `tidewave` (only: :dev) and `ash_ai` (which transitively brings in
`ash_json_api`) are both present, Mix reports a `:only` option conflict on
the `plug` dependency. Adding an explicit `{:plug, "~> 1.19"}` without an
`:only` restriction resolves this.

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
`if code_reloading?` block:

```elixir
defmodule MyAppWeb.Endpoint do
  use Phoenix.Endpoint, otp_app: :my_app_web

  # ... existing plugs (session, static, etc.)

  # Tidewave MCP server (dev only)
  if Code.ensure_loaded?(Tidewave) do
    plug Tidewave
  end

  if code_reloading? do
    socket "/phoenix/live_reload/socket", Phoenix.LiveReloader.Socket
    plug Phoenix.LiveReloader
    plug Phoenix.CodeReloader
    plug Phoenix.Ecto.CheckRepoStatus, otp_app: :my_app
    plug AshAi.Mcp.Dev, otp_app: :my_app
  end

  plug MyAppWeb.Router
end
```

The conditional `Code.ensure_loaded?/1` check ensures Tidewave is only loaded in dev
where the dependency is available.

`AshAi.Mcp.Dev` goes inside the `code_reloading?` block — this is the conventional
location for dev-only plugs in Phoenix endpoints, and the block is positioned before
body parsers in the standard pipeline.

## 4. Domain MCP Router Setup

Add a `/dev/mcp` scope in `router.ex` inside the existing
`if Application.compile_env(:my_app, :dev_routes)` guard:

```elixir
# Inside the existing dev_routes guard block
scope "/dev/mcp" do
  forward "/", AshAi.Mcp.Router,
    tools: @mcp_tools,
    otp_app: :my_app,
    actor: %AshAi{}
end
```

`%AshAi{}` is a struct provided by the `ash_ai` package that acts as an unrestricted
actor — it bypasses all policy checks. This is safe because the route only exists when
`dev_routes` is enabled.

### Domain Tool Declarations

Tools are declared in Ash domains using `tools do` blocks. The domain declares what
tools exist; the router controls which are exposed via a `@mcp_tools` module attribute.

```elixir
# In the Ash domain (e.g., MyApp.Planning)
use Ash.Domain, otp_app: :my_app, extensions: [AshAi]

tools do
  tool(:lookup_items, MyApp.Planning.Item, :read,
    description: "Look up planning items by name or status."
  )
end
```

```elixir
# In router.ex — module attribute listing exposed tools
@mcp_tools [
  :lookup_items
]
```

### MCP Resources (optional)

Domains can also expose browsable data via `mcp_resources`:

```elixir
mcp_resources do
  mcp_resource(:active_items, "myapp://items/active",
    MyApp.Planning.Item, :mcp_active_items,
    title: "Active Items",
    description: "All currently active items.",
    mime_type: "application/json"
  )
end
```

## 5. MCP Configuration

Create `.mcp.json` in the project root:

```json
{
  "mcpServers": {
    "tidewave": {
      "type": "http",
      "url": "http://localhost:4000/tidewave/mcp"
    },
    "my_app": {
      "type": "http",
      "url": "http://localhost:4000/dev/mcp"
    }
  }
}
```

Adjust the port if the Phoenix app uses a non-standard port. Omit the domain MCP
entry if no Ash domains with resources exist.

The domain MCP at `/dev/mcp` is only available when `dev_routes` is enabled. The
`.mcp.json` is only used by Claude Code during local development, so this is fine.

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

The domain MCP provides access to whatever tools are declared in your Ash domains
(e.g., looking up resources, triggering actions).

## 6. CLAUDE.md Template

The CLAUDE.md file has two parts:
1. **Hand-written sections** (project-specific guidance)
2. **Generated sections** (appended by `mix usage_rules.sync`)

### Hand-written sections for all projects:

```markdown
## Ash First

Always use Ash concepts. Never use Ecto directly. Think hard about the
"Ash way" to do things. If unsure, consult the ash-framework skill or
search docs with `mix usage_rules.search_docs`.

## Code Generation

Use `mix igniter.install` and Ash generators as a starting point wherever
possible. Modify generated code rather than writing from scratch.

## MCP Usage

When the Phoenix server is running, prefer MCP tools over reading files
for understanding application state:
- Use domain MCP tools to look up existing resources, actions, and
  relationships before writing new code
- Use Tidewave `project_eval` to test expressions against the running app
- Use Tidewave `get_ash_resources` / `get_ecto_schemas` to discover
  what's already defined

## Feedback Loop

After every change:
1. Run `mix compile` — fix all errors before proceeding
2. Run `mix format --check` — fix formatting
3. Run `mix credo --strict` — address all warnings
4. Run `mix test` for affected modules
5. Use Tidewave `project_eval` to verify runtime behaviour
6. Use domain MCP tools to verify resource/domain state

If an MCP call fails or times out, the Phoenix server may not be running.
Remind the user to start it with `mix phx.server`.

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

## 7. Generation

After all configuration is in place:

```bash
mix deps.get
mix usage_rules.sync --yes
```

**Important:** The task is `usage_rules.sync`, NOT `usage_rules.gen`. The `--yes`
flag auto-accepts igniter's confirmation prompt, which is required for non-interactive
execution (igniter will error on EOF if it tries to prompt without `--yes`).

This generates:
- Usage rules content appended to CLAUDE.md (between `<!-- usage-rules-start -->` and `<!-- usage-rules-end -->` markers)
- Skill files in `.claude/skills/` based on the `build` config

## 8. External Skills

Install the logging best practices skill:

```bash
npx skills add https://github.com/boristane/agent-skills --skill logging-best-practices --yes
```

**Important:** The `--yes` flag auto-selects default agents. Without it, the CLI
prompts interactively for which agents to install to, which will hang in
non-interactive execution.

This provides Claude with guidance on the wide events / canonical log lines pattern
referenced in the CLAUDE.md Logging section.

## 9. Browser Testing Setup (Optional)

If the user opted in, generate a project-specific browser testing skill at
`.claude/skills/browser-testing/SKILL.md`. This complements the generic
`browser-testing-walkthrough` external skill (installed in step 8) with
project-specific context.

### Template

````markdown
---
name: browser-testing
description: Use when manually testing <app_name> features in the browser with GIF recording.
---

# Browser Testing — <AppName>

## Navigation Map

| Page | URL | Key Elements |
|------|-----|--------------|
| Home | http://localhost:<port>/ | Main landing page |
| Dev Dashboard | http://localhost:<port>/dev/dashboard | Phoenix LiveDashboard |
| Dev Mailbox | http://localhost:<port>/dev/mailbox | Email preview |
| Dev MCP | http://localhost:<port>/dev/mcp | MCP server (verify with curl) |

_Add your application-specific routes here._

## Seed Data Reference

Fill in your test users and seed data here. List names, roles, and any
notable state (e.g., overallocated users, edge cases).

Example format:
| User | Role | Notable State |
|------|------|---------------|
| Alice | Admin | Full access |
| Bob | Member | Limited permissions |

## Tool Quick Reference

| Action | Tool | Example |
|--------|------|---------|
| Go to URL | `navigate` | `navigate "http://localhost:<port>/"` |
| Click element | `computer` | `left_click` at coordinates |
| Capture frame | `screenshot` | Each screenshot = 1 GIF frame |
| Zoom detail | `computer` | `region` + `zoom` (NOT a GIF frame) |
| Find target | `find` | Natural language description |
| Type text | `type` | Text input into focused field |

## GIF Budget

- **50 frames max** per recording
- **8-10 frames** per flow for context
- **3-5 flows** total per session
- Use `zoom` freely — it doesn't consume frames

## PR Comment Template

```markdown
## Browser Walkthrough

**What was tested:**
- [ ] Flow 1: [describe what you navigated and verified]
- [ ] Flow 2: [describe what you navigated and verified]

**Summary:** [Written description of what was validated and any issues found.
Reviewers must understand what was tested without watching the GIF.]

![Walkthrough](path/to/recording.gif)
```
````

Replace `<app_name>`, `<AppName>`, and `<port>` with detected values.

## 10. Dev Environment (Optional)

All three options use direnv for shell integration.

### devenv

Generate `.envrc`:
```bash
eval "$(devenv direnvrc)"
use devenv
```

Document requirements for `devenv.nix`:
- Packages: Elixir (>= 1.18), Erlang (>= 27), Node.js, PostgreSQL
- Add `dotenv.enable = true` for `.env` file support
- Configure PostgreSQL port to 5433 to avoid system conflicts
- User looks up devenv syntax for package/service declarations

### flake

Generate `.envrc`:
```bash
use flake
dotenv_if_exists
```

Document requirements for `flake.nix`:
- Packages in flake outputs: Elixir (>= 1.18), Erlang (>= 27), Node.js, PostgreSQL
- `dotenv_if_exists` handles `.env` loading (Nix flakes have no built-in support)
- Configure PostgreSQL port to 5433 to avoid system conflicts
- User looks up Nix syntax for package declarations

### devbox

Generate `.envrc`:
```bash
use devbox
```

devbox auto-loads `.env` from the project root.

#### Add PostgreSQL package

```bash
devbox add postgresql
```

If devbox warns about "legacy format", run `devbox update` to migrate the config.

#### Configure custom port

Port 5432 is commonly used by a system PostgreSQL. Use port 5433
to avoid conflicts. Configure via the `env` section in `devbox.json`:

```json
{
  "packages": [
    "elixir@1.18.1",
    "erlang@27.2",
    "nodejs@25.8.0",
    "postgresql@latest"
  ],
  "env": {
    "PGPORT": "5433",
    "PGHOST": ".devbox/virtenv/postgresql"
  },
  "shell": {
    "init_hook": [
      "if [ ! -d \"$PGDATA\" ]; then initdb && createuser -h $PGHOST -p $PGPORT -s postgres && psql -h $PGHOST -p $PGPORT -U $(whoami) -d postgres -c \"ALTER USER postgres WITH PASSWORD 'postgres';\"; fi"
    ],
    "scripts": {
      "db:start": "devbox services start postgresql",
      "db:stop": "devbox services stop postgresql"
    }
  }
}
```

The `init_hook` automatically bootstraps the database on first `devbox shell` entry:
- Runs `initdb` if the data directory doesn't exist
- Creates a `postgres` superuser role
- Sets the password to `postgres` (matching Phoenix dev defaults)

#### Override process-compose for PostgreSQL

Create a root `process-compose.yml` to override with custom settings:

```yaml
version: "0.5"

processes:
  postgresql:
    command: "pg_ctl start -o \"-k $PGHOST -p $PGPORT\""
    is_daemon: true
    shutdown:
      command: "pg_ctl stop -m fast"
    availability:
      restart: "always"
    readiness_probe:
      exec:
        command: "pg_isready -h $PGHOST -p $PGPORT"
```

**Critical:** The PostgreSQL socket directory (`-k` flag) MUST point to the devbox
virtenv path. The default `/run/postgresql/` is not writable by non-root users.

#### Update Phoenix dev config

Add the matching port to `config/dev.exs`:

```elixir
config :my_app, MyApp.Repo,
  username: "postgres",
  password: "postgres",
  hostname: "localhost",
  port: 5433,
  database: "my_app_dev",
  # ...
```

#### Usage

```bash
devbox shell              # enters shell, runs init_hook on first use
devbox run db:start       # starts PostgreSQL on port 5433
mix ecto.create           # creates the database
mix phx.server            # starts the Phoenix app
```

## 11. Verification Checklist

After setup, confirm:

- [ ] `mix deps.get` succeeds
- [ ] `mix compile` succeeds with no warnings
- [ ] `mix format --check-formatted` passes
- [ ] `CLAUDE.md` exists with hand-written + generated sections
- [ ] `.claude/skills/ash-framework/` directory exists with generated skill
- [ ] `.mcp.json` exists with Tidewave config
- [ ] `logging-best-practices` skill is installed
- [ ] (If devbox) `devbox.json` has PostgreSQL package, env vars, init_hook, and scripts
- [ ] (If devbox) `process-compose.yml` exists with custom port/socket config
- [ ] (If devbox) `config/dev.exs` has matching PostgreSQL port
- [ ] (If direnv) `.envrc` exists
- [ ] Starting the Phoenix server (`mix phx.server`) makes Tidewave available at the configured URL
