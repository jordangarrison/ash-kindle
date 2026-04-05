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
| `plug` | `~> 1.19` | root (umbrella) or app (resolves tidewave/ash_json_api conflict) | all envs |
| `ash` | `~> 3.0` | core app (umbrella) or app | all envs |
| `ash_phoenix` | `~> 2.0` | web app (umbrella) or app | all envs |
| `ash_postgres` | `~> 2.0` | core app (umbrella) or app | all envs |
| `ash_ai` | `~> 0.5` | core app (umbrella) or app | all envs |
| `usage_rules` | `~> 1.2` | root (umbrella) or app | `:dev` only, `runtime: false` |
| `tidewave` | `~> 0.5` | web app (umbrella) or app | `:dev` only |

**Note:** The explicit `plug` dep is required to resolve a `:only` env conflict
between `tidewave` (only: :dev) and `ash_json_api` (transitive via `ash_ai`).

## MCP Servers

### Tidewave

- Port: whatever Phoenix is configured to use (default `4000`)
- MCP URL: `http://localhost:<port>/tidewave/mcp`

### Domain MCP (default: yes if Ash domains detected)

- Dev-only endpoint at `/dev/mcp`
- Uses `AshAi.Mcp.Dev` plug in `endpoint.ex` (inside `code_reloading?` block)
- Uses `AshAi.Mcp.Router` in `router.ex` (inside `dev_routes` guard)
- Actor: `%AshAi{}` (bypasses auth for dev use)
- MCP URL: `http://localhost:<port>/dev/mcp`

### .mcp.json (with both servers)

```json
{
  "mcpServers": {
    "tidewave": {
      "type": "http",
      "url": "http://localhost:<port>/tidewave/mcp"
    },
    "<app_name>": {
      "type": "http",
      "url": "http://localhost:<port>/dev/mcp"
    }
  }
}
```

## CLAUDE.md Sections

The following sections are always included:

1. **Ash First**
2. **Code Generation**
3. **MCP Usage** (prefer MCP tools for app state, use domain MCP before writing code)
4. **Feedback Loop** (compile, format, credo, test, Tidewave eval, domain MCP verify + server-not-running guidance)
5. **Logging** (wide events pattern)

For umbrella projects, also include:

6. **Umbrella Structure**

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
npx skills add https://github.com/boristane/agent-skills --skill logging-best-practices --yes
```

## Dev Environment

Default: **skip** if no config file detected; otherwise default to detected tool.

### `.envrc` Templates

**devenv:**
```bash
eval "$(devenv direnvrc)"
use devenv
```
`.env` loading: via devenv `dotenv.enable = true` in `devenv.nix` (document as a requirement).

**flake:**
```bash
use flake
dotenv_if_exists
```
`.env` loading: via direnv `dotenv_if_exists`.

**devbox:**
```bash
use devbox
```
`.env` loading: built-in (devbox auto-loads `.env` from project root).

### Required Packages (all options)

- Elixir >= 1.18
- Erlang >= 27
- Node.js
- PostgreSQL (custom port 5433 recommended to avoid system conflicts)

### devbox Full Config

When devbox is detected or opted in, default PostgreSQL config:

- **Port:** `5433` (avoids conflict with system PostgreSQL on 5432)
- **Socket directory:** `.devbox/virtenv/postgresql`
- **init_hook:** auto-runs `initdb`, creates `postgres` role, sets password on first shell
- **process-compose.yml:** overrides plugin default with custom port + socket path
- **Convenience scripts:** `db:start`, `db:stop`
