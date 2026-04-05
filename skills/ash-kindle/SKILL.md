---
name: ash-kindle
description: Set up AI-assisted dev tooling (usage_rules, Tidewave, Ash AI, CLAUDE.md, skills, logging) for Ash/Phoenix projects. Use when creating a new Ash/Phoenix project or adding AI tooling to an existing one.
---

# Ash Kindle — AI Dev Environment Setup for Ash/Phoenix

You are setting up the AI-assisted development environment for an Ash/Phoenix project.
This includes: usage_rules, Tidewave MCP, Ash AI, CLAUDE.md, Claude Code skills, the
logging-best-practices skill, and optionally direnv/Nix.

## Step 0: Read References

Before doing anything, read the full setup guide and defaults:

```
Read skills/ash-kindle/references/setup-guide.md
Read skills/ash-kindle/defaults.md
```

## Step 1: Detect Project State

Determine:
- Is this a new or existing project? (check for `mix.exs` in the directory)
- Is it an umbrella or single app?
- What is the OTP app name? (e.g., `my_app`)
- What is the web app module? (e.g., `MyAppWeb`)
- What Ash domains exist? (check `ash_domains` in config)
- What Phoenix port is configured? (default 4000)
- Are there domain-specific deps beyond Ash? (Oban, Req, LangChain, etc.)
- Which dev environment tool is in use? (check for `devenv.nix`, `flake.nix`, or `devbox.json`)
- Is domain MCP already configured? (check for `AshAi.Mcp.Dev` in `endpoint.ex`, `"/dev/mcp"` in `router.ex`, and a non-tidewave entry in `.mcp.json` — if all three present, skip; if partially configured, warn the user and offer to complete setup)
- Is browser testing already set up? (check for `.claude/skills/browser-testing/SKILL.md`)

Use `mix.exs`, `config/config.exs`, and the project file structure to answer these.

### If no project exists yet

If the directory has no `mix.exs`, offer to create a new Phoenix project:

1. Install the Phoenix generator: `mix archive.install hex phx_new --force`
2. Create the project: `yes | mix phx.new . --app <app_name>`
   - The `yes |` prefix is needed to auto-confirm the existing directory prompt
3. Ask about structure (single app recommended — can migrate to umbrella later)
4. Then proceed with the rest of the setup

## Step 2: Ask or Accept Defaults

Present the user with what you'll set up:

> I'll configure the following AI dev tooling for your Ash/Phoenix project:
>
> 1. **usage_rules** — CLAUDE.md generation + doc search from deps
> 2. **Tidewave** — dev MCP server for runtime introspection (eval, logs, schemas)
> 3. **Domain MCP** — dev-only MCP server exposing your Ash domain tools (requires Ash domains with resources)
> 4. **Ash AI** — usage rules and MCP tooling for Ash resources
> 5. **CLAUDE.md** — project instructions (Ash First, MCP Usage, Feedback Loop, Logging)
> 6. **Claude Code skills** — auto-generated from usage_rules config
> 7. **logging-best-practices** — external skill for wide events pattern
> 8. **Browser testing** — optional GIF-recorded browser walkthrough skill (requires Claude-in-Chrome)
> 9. **Dev environment** — optional setup for devenv, flake, or devbox with PostgreSQL
>
> Want to customize any of these, or are defaults fine?

If the user says **defaults are fine**, proceed with all defaults from `defaults.md`.
Default for Domain MCP: **yes** if Ash domains with resources are detected, **no** otherwise.
Default for Browser testing: **no** (opt-in only).
Default for Dev environment: **skip** if no config file detected, otherwise default to detected tool.

If the user wants to **customize**, ask about each piece one at a time:
- Which usage_rules to include (`:elixir`, `:otp`, `:phoenix`, others?)
- Which skills to build (default: `ash-framework`; suggest others based on detected deps)
- Whether to set up domain MCP (and which domain to wire first)
- Whether to set up browser testing
- Which dev environment tool to use (devenv, flake, or devbox)
- Any additional CLAUDE.md sections

## Step 3: Install Dependencies

### Known issue: `plug` dependency conflict

When `tidewave` (only: :dev) and `ash_ai` (which brings in `ash_json_api`) are
both present, there is a `:only` env conflict on the `plug` dependency. **You must
add an explicit `plug` dependency** to resolve this:

```elixir
{:plug, "~> 1.19"}
```

### For umbrella projects (root mix.exs):

```elixir
# Add to deps in root mix.exs
{:plug, "~> 1.19"},
{:usage_rules, "~> 1.2", only: :dev, runtime: false}
```

### For the web app (or single app) mix.exs:

```elixir
# Add to deps
{:tidewave, "~> 0.5", only: :dev}
```

### For the core app (or single app) mix.exs:

```elixir
# Add to deps
{:ash_ai, "~> 0.5"}
```

### For single app projects, all deps in one `mix.exs`:

```elixir
{:plug, "~> 1.19"},
{:ash, "~> 3.0"},
{:ash_phoenix, "~> 2.0"},
{:ash_postgres, "~> 2.0"},
{:ash_ai, "~> 0.5"},
{:usage_rules, "~> 1.2", only: :dev, runtime: false},
{:tidewave, "~> 0.5", only: :dev}
```

Then run:
```bash
mix deps.get
```

## Step 4: Configure usage_rules

Add the `usage_rules()` function and reference it in the project config.
See `defaults.md` for the default configuration. Adapt based on:

- **Umbrella vs single app:** In umbrella, `usage_rules` goes in root mix.exs
- **Detected deps:** If Oban, Req, LangChain etc. are present, suggest additional skill builds
- **Skills location:** Always `.claude/skills`

In the `project/0` function, add:
```elixir
listeners: [Phoenix.CodeReloader],
usage_rules: usage_rules()
```

## Step 5: Wire Up MCP Servers

### Tidewave

In the web app's `endpoint.ex`, add the Tidewave plug **before** the `if code_reloading?` block:

```elixir
if Code.ensure_loaded?(Tidewave) do
  plug Tidewave
end
```

### Domain MCP (if Ash domains with resources exist)

The domain MCP requires two pieces. See `references/setup-guide.md` for full details.

**1. Endpoint plug:** Add `AshAi.Mcp.Dev` inside the `if code_reloading?` block in `endpoint.ex`, after `Phoenix.CodeReloader`:

```elixir
if code_reloading? do
  socket "/phoenix/live_reload/socket", Phoenix.LiveReloader.Socket
  plug Phoenix.LiveReloader
  plug Phoenix.CodeReloader
  plug Phoenix.Ecto.CheckRepoStatus, otp_app: :my_app
  plug AshAi.Mcp.Dev, otp_app: :my_app
end
```

**2. Router scope:** Add a `/dev/mcp` scope inside the `if Application.compile_env(:my_app, :dev_routes)` guard in `router.ex`:

```elixir
scope "/dev/mcp" do
  forward "/", AshAi.Mcp.Router,
    tools: @mcp_tools,
    otp_app: :my_app,
    actor: %AshAi{}
end
```

**3. Domain tools:** Add a `tools do` block in the Ash domain, and a `@mcp_tools` module attribute in the router listing exposed tool names. Ask the user which domain to wire first.

If no Ash domains with resources exist yet, skip the domain MCP and just set up Tidewave.

### .mcp.json

Create `.mcp.json` in the project root. Include both servers if domain MCP was set up:

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

Replace `<port>` with the detected Phoenix port (default 4000) and `<app_name>` with the OTP app name. If domain MCP was skipped, omit the `<app_name>` entry.

## Step 6: Write CLAUDE.md

Write the hand-written sections of CLAUDE.md. See `references/setup-guide.md` for the
full template. The key sections are:

- **Ash First** — always use Ash concepts, never raw Ecto
- **Code Generation** — use igniter and Ash generators
- **MCP Usage** — prefer MCP tools over file reading for app state; use domain MCP to look up resources before writing code
- **Umbrella Structure** — (if umbrella) which app owns what
- **Feedback Loop** — compile, format, credo, test, tidewave eval, domain MCP verify; note that MCP failures likely mean Phoenix server isn't running
- **Logging** — wide events / canonical log lines pattern

Leave space after the hand-written sections for usage_rules to append its generated content.

## Step 7: Generate Skills and CLAUDE.md Content

```bash
mix usage_rules.sync --yes
```

**Important:** The task is `usage_rules.sync`, not `usage_rules.gen`. The `--yes`
flag is required for non-interactive execution (otherwise igniter prompts for
confirmation on large diffs).

This generates:
- The usage_rules sections appended to CLAUDE.md
- Claude Code skill files in `.claude/skills/`

Verify the generated files look correct.

## Step 8: Install External Skills

```bash
npx skills add https://github.com/boristane/agent-skills --skill logging-best-practices --yes
```

If the user opted in to browser testing:

```bash
npx skills add https://github.com/jordangarrison/superpowers --skill browser-testing-walkthrough --yes
```

**Important:** The `--yes` flag is required for non-interactive execution (otherwise
it prompts for which agents to install to).

## Step 9: Optional — Browser Testing Setup

If the user opted in to browser testing (Step 2), generate a project-specific browser
testing skill. The external skill (installed in Step 8) handles the generic workflow;
this skill provides project-specific context.

Create `.claude/skills/browser-testing/SKILL.md` with these sections:

1. **Navigation Map** — table with Page, URL, Key Elements columns. Pre-fill with
   detected routes (at minimum: `/`, `/dev/dashboard`, `/dev/mailbox`). Use the
   detected Phoenix port.
2. **Seed Data Reference** — placeholder: "Fill in your test users and seed data here.
   List names, roles, and any notable state (e.g., overallocated users, edge cases)."
3. **Tool Quick Reference** — condensed cheat sheet: navigate, click, screenshot,
   zoom, find, type
4. **GIF Budget** — 50 frames max, 8-10 per flow, 3-5 flows total
5. **PR Comment Template** — requires a written summary of what was tested alongside
   the GIF embed. Reviewers must understand what was validated without watching the GIF.

See `references/setup-guide.md` for the full template.

## Step 10: Optional — Dev Environment

If the user opted in (or a dev environment config file was detected in Step 1), set
up the dev environment. All three options use direnv.

Ask: "Which dev environment tool do you use?" (default to detected tool):
- **devenv** — `devenv.nix` + direnv
- **flake** — `flake.nix` + direnv
- **devbox** — `devbox.json` + direnv

### For all options:

1. Generate the appropriate `.envrc` (see `defaults.md` for templates)
2. Document required packages: Elixir (>= 1.18), Erlang (>= 27), Node.js, PostgreSQL
3. Note custom PostgreSQL port (5433 to avoid system conflicts)
4. Update `config/dev.exs` with matching port if needed

### Per-option details:

- **devenv:** Generate `.envrc` only. Document that the user needs Elixir, Erlang, Node.js, PostgreSQL services in their `devenv.nix`, and should add `dotenv.enable = true` for `.env` support. User looks up devenv-specific syntax.
- **flake:** Generate `.envrc` only. Document that the user needs Elixir, Erlang, Node.js, PostgreSQL in their flake outputs. User looks up Nix-specific syntax.
- **devbox:** Generate full `devbox.json`, `process-compose.yml`, and `.envrc`. See `references/setup-guide.md` for the complete devbox config with PostgreSQL, init_hook, and convenience scripts.

## Step 11: Verify

```bash
mix deps.get
mix compile
mix format --check-formatted
```

Confirm:
- `mix compile` succeeds
- CLAUDE.md has both hand-written and generated sections
- `.claude/skills/` contains generated skill files
- `.mcp.json` exists with Tidewave config
- Tidewave is accessible when the Phoenix server is running

Report what was set up and any manual steps remaining.
