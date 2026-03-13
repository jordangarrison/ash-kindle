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
- Is this a new or existing project?
- Is it an umbrella or single app?
- What is the OTP app name? (e.g., `my_app`)
- What is the web app module? (e.g., `MyAppWeb`)
- What Ash domains exist? (check `ash_domains` in config)
- What Phoenix port is configured? (default 4000)
- Are there domain-specific deps beyond Ash? (Oban, Req, LangChain, etc.)

Use `mix.exs`, `config/config.exs`, and the project file structure to answer these.

## Step 2: Ask or Accept Defaults

Present the user with what you'll set up:

> I'll configure the following AI dev tooling for your Ash/Phoenix project:
>
> 1. **usage_rules** — CLAUDE.md generation + doc search from deps
> 2. **Tidewave** — dev MCP server for runtime introspection (eval, logs, schemas)
> 3. **Ash AI** — usage rules and MCP tooling for Ash resources
> 4. **CLAUDE.md** — project instructions (Ash First, Feedback Loop, Logging)
> 5. **Claude Code skills** — auto-generated from usage_rules config
> 6. **logging-best-practices** — external skill for wide events pattern
> 7. **direnv/.envrc** — optional Nix/direnv setup
>
> Want to customize any of these, or are defaults fine?

If the user says **defaults are fine**, proceed with all defaults from `defaults.md`.

If the user wants to **customize**, ask about each piece one at a time:
- Which usage_rules to include (`:elixir`, `:otp`, `:phoenix`, others?)
- Which skills to build (default: `ash-framework`; suggest others based on detected deps)
- Whether to include direnv/Nix setup
- Any additional CLAUDE.md sections

## Step 3: Install Dependencies

### For umbrella projects (root mix.exs):

```elixir
# Add to deps in root mix.exs
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

## Step 5: Wire Up Tidewave

In the web app's `endpoint.ex`, add the Tidewave plug **before** the router plug:

```elixir
if Code.ensure_loaded?(Tidewave) do
  plug Tidewave
end
```

Create `.mcp.json` in the project root:

```json
{
  "mcpServers": {
    "tidewave": {
      "type": "http",
      "url": "http://localhost:<port>/tidewave/mcp"
    }
  }
}
```

Replace `<port>` with the detected Phoenix port (default 4000).

## Step 6: Write CLAUDE.md

Write the hand-written sections of CLAUDE.md. See `references/setup-guide.md` for the
full template. The key sections are:

- **Ash First** — always use Ash concepts, never raw Ecto
- **Code Generation** — use igniter and Ash generators
- **Umbrella Structure** — (if umbrella) which app owns what
- **Feedback Loop** — compile, format, credo, test, tidewave eval, ash ai verify
- **Logging** — wide events / canonical log lines pattern

Leave space after the hand-written sections for usage_rules to append its generated content.

## Step 7: Generate Skills and CLAUDE.md Content

```bash
mix usage_rules.gen
```

This generates:
- The usage_rules sections appended to CLAUDE.md
- Claude Code skill files in `.claude/skills/`

Verify the generated files look correct.

## Step 8: Install External Skills

```bash
npx skills add https://github.com/boristane/agent-skills --skill logging-best-practices
```

## Step 9: Optional — direnv/Nix Setup

If the user opted in, create `.envrc`:

```bash
use flake
dotenv_if_exists
```

And note that they'll need a `flake.nix` appropriate for their Elixir/Phoenix setup.

## Step 10: Verify

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
