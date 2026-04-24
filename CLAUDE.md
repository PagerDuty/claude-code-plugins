# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Overview

This is a Claude Code plugin marketplace for PagerDuty integrations. It contains plugins that bring operational intelligence into development workflows by correlating code changes with PagerDuty incident data.

**Current plugins:**
- `pagerduty`: Includes multiple commands:
  - Pre-commit risk scoring using PagerDuty incident history and git diff analysis
  - PagerDuty skill creation and management for AI agents

## Repository Structure

```
.claude-plugin/           # Marketplace and plugin metadata
  marketplace.json        # Marketplace registry definition
  plugin.json            # Plugin metadata (name, version, author)
commands/                # Plugin slash commands
  pre-commit-risk-scoring.md   # Risk assessment command
  create-pagerduty-skill.md    # Skill creation/management command
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
`.mcp.json` declares external MCP servers the plugins depend on. The PagerDuty server is an HTTP MCP server that requires `PAGERDUTY_API_KEY` to be set in the environment (via `.claude/settings.local.json` or `~/.claude/settings.json`).

## Development Workflow

### Testing Changes Locally

After modifying plugin files, test the changes by installing the marketplace from your local clone:

```bash
# Add your local clone as a marketplace
/plugin marketplace add /Users/mmayo/repos/claude-code-plugins

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
4. **Configuration persistence**: Cache resolved service mappings to `.claude/risk-config.json` to avoid repeated lookups, but NEVER cache when an explicit argument override is provided
5. **Parallel execution**: Independent API calls (active incidents + incident history + change events) and git operations should run in parallel
6. **Structured output**: Use consistent formatting with clear sections, risk scores, and actionable recommendations

### Skill Creation Command (create-pagerduty-skill)

The `create-pagerduty-skill.md` command provides an interactive workflow for creating and updating PagerDuty skills via API:

1. **Pre-flight checks**: Uses `ToolSearch` to verify PagerDuty Advance MCP is available before proceeding. Fails explicitly with setup instructions if unavailable.
2. **Mode selection**: Supports both creating new skills and updating existing skills
3. **Scope selection**: REQUIRED - asks user to choose between "account" (shared/team-level) or "user" (personal/individual) scope
4. **Full replace awareness**: UPDATE mode fetches current skill first to preserve all fields (update_skill_tool is a full replacement operation)
5. **API constraints**: Validates name format (kebab-case, max 60 chars), description (max 1024 chars), and instructions (max 5000 tokens)
6. **SRE Agent only**: Currently only the SRE Agent supports skills - agent_type is always "sre"
7. **Skill limit handling**: Account scope allows 50 skills per agent, user scope allows 25 - prompts user to update or delete when limit reached
8. **Scope immutability**: Scope and name cannot be changed after creation (must delete and recreate to change scope)
9. **Immediate deployment**: Skills are available immediately after API call (no S3 upload needed)

**Critical patterns:**
- Never proceed without MCP access - no silent degradation
- Always hard-code agent_type to "sre" (only SRE Agent supports skills currently)
- Always ask for scope before any operations - scope is required for all API calls
- Same skill name can exist in both scopes independently - identity is (agent, scope, name)
- Always validate name uniqueness within the chosen scope via `list_skills_tool`
- Estimate token count for instructions (chars / 4 ≈ tokens) and warn at 90% of limit
- For UPDATE mode, fetch current skill and pre-fill all prompts with existing values
- Optional JSON backup only on user request to avoid repository clutter

### PagerDuty MCP Tool Names

When referencing PagerDuty MCP tools in command `allowed-tools` lists, use the full plugin-scoped name:

**Plugin PagerDuty (service/incident operations):**
- `mcp__plugin_pagerduty_pagerduty__get_service`
- `mcp__plugin_pagerduty_pagerduty__list_services`
- `mcp__plugin_pagerduty_pagerduty__list_incidents`
- `mcp__plugin_pagerduty_pagerduty__list_incident_notes`
- `mcp__plugin_pagerduty_pagerduty__list_service_change_events`

**PagerDuty Advance MCP (skills management):**
- `mcp__pagerduty-advance-mcp-__create_skill_tool`
- `mcp__pagerduty-advance-mcp-__get_skill_tool`
- `mcp__pagerduty-advance-mcp-__list_skills_tool`
- `mcp__pagerduty-advance-mcp-__update_skill_tool`
- `mcp__pagerduty-advance-mcp-__delete_skill_tool`

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
2. **Cached config** in `.claude/risk-config.json`
3. **Backstage catalog** `catalog-info.yaml` with `pagerduty.com/service-id` annotation
4. **Auto-detection** from repository name
5. **User prompt** as last resort

**Critical**: When validating a Backstage service ID, call `get_service` with the literal ID. Do NOT pass UUIDs to `list_services` - the API returns a 502 error.

### Incident Fetching Strategy

The pre-commit command makes parallel calls for:
- High-urgency incidents (last 90 days)
- Low-urgency incidents (last 90 days)
- Service change events

The PagerDuty incidents API caps results at 1000 per call. When exactly 1000 incidents are returned, note the history is partial and report the actual date range covered.

### Risk Scoring Model

Scores range from 0-5 based on:
- **Active incidents** on the service (highest weight)
- **Incident correlation** with changed files/areas
- **Structural risk signals** (auth, migrations, config, dependencies, infrastructure)
- **Change magnitude** (files affected, lines changed, directory spread)
- **Pattern similarity** to changes that preceded past incidents

**Important**: Adjust scoring for "noisy alerts" - if incident volume is dominated by a single repeating alert, weight those incidents lower than diverse incident patterns.
