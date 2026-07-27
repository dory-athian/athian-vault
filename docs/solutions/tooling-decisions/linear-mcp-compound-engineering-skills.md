---
title: Connecting Compound Engineering Skills to Linear via MCP
date: 2026-06-23
category: docs/solutions/tooling-decisions/
module: claude-code-skills
problem_type: tooling_decision
component: development_workflow
severity: medium
applies_when:
  - migrating from Jira to Linear (partially or fully)
  - using compound-engineering skills like /ce-work, /ce-code-review, /lfg
  - skills are creating Jira tickets when Linear is expected
  - adding a new issue tracker alongside an existing one
  - piloting Linear on a subset of projects before full migration
symptoms:
  - compound-engineering skills file tickets in Jira when Linear is intended
  - /ce-work and related skills do not surface Linear as an option
  - Linear issues never created despite expecting them
root_cause: incomplete_setup
resolution_type: config_change
related_components:
  - tooling
  - documentation
tags:
  - linear
  - jira
  - mcp
  - issue-tracker
  - compound-engineering
  - claude-code
  - migration
  - configuration
---

# Connecting Compound Engineering Skills to Linear via MCP

## Context

The compound-engineering Claude Code skills (`/ce-work`, `/ce-code-review`, `/lfg`, etc.) file tickets in whichever issue tracker they detect at runtime. Detection has two parts: reading CLAUDE.md for a tracker reference, and probing for a live MCP tool. If either is missing, the skills fall back to GitHub Issues via `gh` or skip filing entirely — silently, with no error.

At Athian, CLAUDE.md listed Jira as the "engineering system of record" and made no mention of Linear. When the compliance team began piloting Linear, the skills had no signal to use it. A prior `/ce-setup` health check confirmed `gh` was available as a fallback — which is why ticket filing appeared to work but silently routed to GitHub Issues instead of Linear. (session history)

## Guidance

To wire up a new issue tracker so compound-engineering skills use it, both signals must be present:

**Step 1: Update CLAUDE.md with an explicit tracker reference.**

Add the tracker by name, its URL or identifier, and what scope it covers. Be specific about which work goes where — the skills read this to set `tracker_name` and decide which sink to target. Vague wording (e.g., "we use Linear") isn't enough; name the tracker and scope clearly.

**Step 2: Add the MCP server to `~/.claude/settings.json` under `mcpServers`.**

The skills probe at runtime for a responsive MCP tool. Without this, even a correctly worded CLAUDE.md causes a fallback. Use `npx -y <package>` so the package installs on first run without a separate install step. Pass credentials as env vars, not inline in args.

**Step 3: If CLAUDE.md is a symlink, edit the real file.**

At Athian, `/Users/doryweiss/Athian/CLAUDE.md` is a symlink to `/Users/doryweiss/dotfiles/CLAUDE-work.md`. Edit the real file directly — some tools follow the symlink and some don't.

## Why This Matters

The fallback chain is: named tracker MCP → GitHub Issues (`gh`) → no sink. A misconfigured or missing setup doesn't throw an error — it just quietly files somewhere else. You won't know the skills are misrouting unless you check where tickets end up.

Both signals must be present. CLAUDE.md alone isn't enough: if the MCP server isn't loaded, the skills treat the named tracker as unavailable and fall back. MCP alone isn't enough either: if CLAUDE.md doesn't name the tracker, the skills don't know to probe for it.

During the Jira → Linear migration period, the CLAUDE.md wording must be precise enough that the skills can distinguish which tracker applies to which work. Ambiguous wording causes all tickets to go to whichever sink responds first.

## When to Apply

- A new issue tracker is introduced (migration, pilot, or parallel use)
- A tracker's scope changes (e.g., Linear expands from Compliance to all teams)
- The compound-engineering skills aren't filing tickets where expected
- A new developer sets up their Claude Code environment and needs to connect to the team's tracker
- A tracker MCP package or API key rotates and needs updating in settings.json

## Examples

**CLAUDE.md wording (Athian, dual-tracker setup):**

```
- **Issue trackers:** Jira (Athian Sustainability, engineering system of record); Linear (linear.app)
  for Athian Compliance — piloting a migration from Jira. All new Compliance work is filed in Linear.
  Use the Linear MCP server to create and manage Linear issues.
```

**`~/.claude/settings.json` — `mcpServers` entry:**

```json
{
  "permissions": { ... },
  "enabledPlugins": [ ... ],
  "extraKnownMarketplaces": [ ... ],
  "mcpServers": {
    "linear": {
      "command": "npx",
      "args": ["-y", "@linear/mcp-server"],
      "env": {
        "LINEAR_API_KEY": "lin_api_..."
      }
    }
  }
}
```

The Linear API key starts with `lin_api_` and is generated at Linear → Settings → API → Personal API keys. Keep it in `settings.json` (which is local and not checked in) — do not commit the key value to version control.

For Jira, the equivalent MCP package and auth differ, but the pattern — name it in CLAUDE.md, wire the MCP server, confirm both are present — is the same.

## Related

- Linear MCP package: `@linear/mcp-server` (official, installed via `npx -y @linear/mcp-server`)
- Compound engineering tracker detection logic: `tracker-defer.md` in the compound-engineering plugin (`skills/ce-work/references/tracker-defer.md`)
