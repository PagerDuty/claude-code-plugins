# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Overview

This is a Claude Code plugin marketplace for PagerDuty integrations. It contains plugins that bring operational intelligence into development workflows by correlating code changes with PagerDuty incident data.

**Current plugins:**
- `pagerduty`: Includes multiple commands:
  - Pre-commit risk scoring using PagerDuty incident history and git diff analysis
  - Incident investigation with the PagerDuty SRE Agent
  - On-call handoff summaries

## Repository Structure

```
.claude-plugin/           # Marketplace and plugin metadata
  marketplace.json        # Marketplace registry definition
  plugin.json            # Plugin metadata (name, version, author)
commands/                # Plugin slash commands
  pre-commit-risk-scoring.md   # Risk assessment command
  investigate.md               # SRE Agent incident investigation command
  on-call-handoff.md           # On-call handoff summary command
.mcp.json               # PagerDuty MCP server configuration
```

## Plugin Architecture

### Marketplace Registration
The marketplace is defined in `.claude-plugin/marketplace.json` with owner metadata and a list of plugins. Each plugin entry must specify:
- `name`: Plugin identifier (e.g., "pagerduty")
- `source`: Path to plugin definition (typically `"./"`)
- `description`: What the plugin does
- `version`: Semantic version string
- `author`: Attribution metadata

### Plugin Metadata
Each plugin has a `plugin.json` at its source path containing name, description, version, and author. **The version in `plugin.json` must match the version in `marketplace.json`.**

### Command Definitions
Commands are markdown files in the `commands/` directory with frontmatter:
- `description`: Human-readable command purpose
- `argument-hint`: Optional syntax hint for arguments
- `allowed-tools`: Whitelist of tools the command can invoke

The markdown body contains the full command prompt that gets executed when the slash command is invoked.

### MCP Server Integration
`.mcp.json` declares a single HTTP MCP server, `pagerduty`, at `https://mcp.pagerduty.com/mcp?tool_groups=full-tools,ai-agents`. It authenticates with OAuth using CIMD (Client ID Metadata Document): Claude Code is a CIMD client, so users need no client ID or secret, and `.mcp.json` and `plugin.json` carry no auth configuration.

- A PagerDuty account admin must install the Claude Code app once per account: https://app.pagerduty.com/oauth-apps/PU4FJQH. Otherwise sign-in fails after login with an "app not installed in the account" error.
- Users sign in from `/mcp`.
- If the app cannot be installed, users can register their own OAuth app and connect with its client ID and secret instead (documented in the README under "Can't get the app installed?"). Every command's error offers this alternative too.
- Orgs that want narrower scopes can register their own Scoped OAuth app instead (documented in the README under "Restricting scopes").

Every command's "MCP unavailable" error must include the admin install link and the docs link: https://docs.pagerduty.com/ai-automation/advance/pagerduty-mcp-server/cimd-clients

## Development Workflow

### Testing Changes Locally

After modifying plugin files, test the changes by installing the marketplace from your local clone:

```bash
# Add your local clone as a marketplace
/plugin marketplace add /path/to/claude-code-plugins

# Install the plugin from your local marketplace
/plugin install pagerduty@pagerduty-claude-code-plugins

# Test the command
/pagerduty:pre-commit-risk-scoring
```

Changes to command markdown files take effect immediately. Changes to metadata files (marketplace.json, plugin.json) require reinstalling the plugin.

### Version Updates

When releasing changes, **both** `.claude-plugin/marketplace.json` and `.claude-plugin/plugin.json` must be updated with the same version number. Mismatched versions will cause marketplace installation issues.

### Command Design Guidelines

The `pre-commit-risk-scoring.md` command demonstrates key patterns:

1. **Tool availability checks**: Always verify required MCP tools are loaded via `ToolSearch` before proceeding
2. **Graceful degradation**: Do NOT silently degrade when required tools are missing - fail explicitly with actionable error messages
3. **Service resolution precedence**: Explicit arguments > cached config > catalog annotations > auto-detection > user prompt
4. **Configuration persistence**: Cache resolved service mappings to `.pagerduty/pagerduty-risk-config.json` (a legacy `.claude/risk-config.json` is migrated on first read) to avoid repeated lookups, but NEVER cache when an explicit argument override is provided
5. **Parallel execution**: Independent API calls (active incidents + incident history + change events) and git operations should run in parallel
6. **Structured output**: Use consistent formatting with clear sections, risk scores, and actionable recommendations

### PagerDuty MCP Tool Names

When referencing PagerDuty MCP tools in command `allowed-tools` lists, use the full plugin-scoped name. There is one server, `pagerduty`, so every tool is `mcp__plugin_pagerduty_pagerduty__<tool>`.

**Consolidated tools** take an `action` parameter (arguments go in a `request` object):
- Read: `browse_incidents` (`list`, `get`, `list_notes`, `list_alerts`, `get_alert`, `context`, `list_change_events`, `list_workflows`, `get_workflow`), `browse_services` (`list`, `get`), `browse_change_events` (`list`, `get`, `list_service`), `browse_users` (`get`, `list`), `browse_escalation_policies` (`list`, `get`), `browse_schedules` (`list`, `get`, `list_users`, `list_oncalls`, ...), `browse_teams`, `browse_activity`, `browse_alert_grouping`, `browse_event_orchestrations`, `browse_status_pages`
- Write: `manage_incidents`, `manage_services`, `manage_schedules`, `manage_teams`, `manage_alert_grouping`, `manage_event_orchestrations`, `manage_status_pages`

Tools used by the commands, for example:
- `mcp__plugin_pagerduty_pagerduty__browse_services` (`list`, `get`)
- `mcp__plugin_pagerduty_pagerduty__browse_incidents` (`list`, `list_notes`)
- `mcp__plugin_pagerduty_pagerduty__browse_change_events` (`list_service`)

**Advance tools (same server, `ai-agents` tool group):**
- `mcp__plugin_pagerduty_pagerduty__paige_sre_agent_send_message` (returns `conversation_id`; AI Actions cost confirmation required once per conversation)
- `mcp__plugin_pagerduty_pagerduty__paige_sre_agent_get_response` (poll until `is_complete`)

## Contributing

- Follow [conventional commits](https://www.conventionalcommits.org) for all PRs and commits
- PR titles must use lowercase and start with `feat:`, `fix:`, `refactor:`, or `chore:`
- All PRs are squash-merged to `main` by maintainers
- Reference related issues in PR descriptions
- Test plugins locally before submitting PRs

## Key Implementation Details

### Service Resolution Chain (pre-commit-risk-scoring)

The service resolution follows a strict priority order:
1. **Explicit argument** (one-time override, not cached)
2. **Cached config** in `.pagerduty/pagerduty-risk-config.json` (legacy `.claude/risk-config.json` is migrated)
3. **Backstage catalog** `catalog-info.yaml` with `pagerduty.com/service-id` annotation
4. **Auto-detection** from repository name
5. **User prompt** as last resort

**Critical**: When validating a Backstage service ID, call `browse_services` with action `get` and the literal ID as `service_id`. Do NOT pass IDs as the `query` of a `browse_services` `list` call - the API returns a 502 error.

### Incident Fetching Strategy

The pre-commit command makes parallel calls for:
- High-urgency incidents (last 90 days)
- Low-urgency incidents (last 90 days)
- Service change events

`browse_incidents` returns at most 100 incidents per call. When a response has `truncated: true`, page with `offset`. Stop after 1000 incidents; if you stop there, note the history is partial and report the actual date range covered.

### Risk Scoring Model

Scores range from 0-5 based on:
- **Active incidents** on the service (highest weight)
- **Incident correlation** with changed files/areas
- **Structural risk signals** (auth, migrations, config, dependencies, infrastructure)
- **Change magnitude** (files affected, lines changed, directory spread)
- **Pattern similarity** to changes that preceded past incidents

**Important**: Adjust scoring for "noisy alerts" - if incident volume is dominated by a single repeating alert, weight those incidents lower than diverse incident patterns.
