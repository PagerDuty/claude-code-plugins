# PagerDuty Claude Code Plugins

[![License](https://img.shields.io/badge/License-Apache%202.0-blue.svg)](https://opensource.org/licenses/Apache-2.0)

PagerDuty's open source [Claude Code plugins](https://code.claude.com/docs/en/plugins) that bring operational intelligence into your development workflow.

## Available plugins

| Plugin                                                    | Description                                                                                                  |
| --------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------ |
| pre-commit-risk-scoring | Pre-commit risk assessment using PagerDuty incident history, change event correlation, and git diff analysis  |
| create-pagerduty-skill | Create and manage PagerDuty skills for AI agents through guided interview (Early Access)  |

## Getting started

### Prerequisites

- [Claude Code](https://code.claude.com) v1.0.33 or later

### 1. Add the marketplace

Run this once in any Claude Code session:

```bash
/plugin marketplace add pagerduty/claude-code-plugins
```

### 2. Browse and install plugins

Open the plugin manager and go to the **Discover** tab to browse available plugins:

```bash
/plugin
```

Or install a plugin directly:

```bash
/plugin install pagerduty@pagerduty-claude-code-plugins
```

### 3. Configure API key

The plugins bundle a PagerDuty MCP server that needs an API token. Git commit history is read directly from the local repository.

**Option A: Per-repo via `.claude/settings.local.json`** (recommended)

This file is gitignored by default, so secrets stay local:

```json
{
  "env": {
    "PAGERDUTY_API_KEY": "your-pagerduty-token"
  }
}
```

If the file already exists, merge the `env` key into it.

**Option B: Global via `~/.claude/settings.json`**

This applies the key to all projects:

```json
{
  "env": {
    "PAGERDUTY_API_KEY": "your-pagerduty-token"
  }
}
```

If the file already exists, merge the `env` key into it.

> **Note:** You may need to restart VS Code for environment variable changes to take effect.

#### Where to get the token

PagerDuty > User Profile > User Settings > API Access > Create API User Token.

Restart Claude Code after setting the key so the MCP server picks it up.

## Set up for your team

You can configure a repository so team members are automatically prompted to add this marketplace when they trust the project folder. Add the following to your project's `.claude/settings.json`:

```json
{
  "extraKnownMarketplaces": {
    "pagerduty-claude-code-plugins": {
      "source": {
        "source": "github",
        "repo": "pagerduty/claude-code-plugins"
      }
    }
  }
}
```

To also auto-enable specific plugins for the project:

```json
{
  "extraKnownMarketplaces": {
    "pagerduty-claude-code-plugins": {
      "source": {
        "source": "github",
        "repo": "pagerduty/claude-code-plugins"
      }
    }
  },
  "enabledPlugins": {
    "pagerduty@pagerduty-claude-code-plugins": true
  }
}
```

## Managing plugins

Update the marketplace to get the latest plugins:

```bash
/plugin marketplace update pagerduty-claude-code-plugins
```

Disable a plugin without uninstalling:

```bash
/plugin disable pagerduty@pagerduty-claude-code-plugins
```

Uninstall a plugin:

```bash
/plugin uninstall pagerduty@pagerduty-claude-code-plugins
```

## Plugin: Pre-Commit Risk Scoring

A plugin that assesses pre-commit risk by correlating PagerDuty incident history with current code changes.

The `/pagerduty:pre-commit-risk-scoring` command gathers PagerDuty incident data for your service, analyzes your current git diff, and looks for correlations between the areas you are changing and areas that have historically caused incidents. It surfaces active incidents, recent incident patterns, structural risk signals in the diff, and actionable recommendations.

### Usage

With uncommitted changes in your working tree:

```bash
/pagerduty:pre-commit-risk-scoring
```

On first run, the plugin resolves your repository to a PagerDuty service through a fallback chain:

1. Explicit argument passed at invocation (one-time override, not cached)
2. Cached config in `.claude/risk-config.json`
3. Backstage `catalog-info.yaml` annotation (`pagerduty.com/service-id`)
4. Auto-detection by matching the repository name against PagerDuty services
5. Manual input via interactive prompt

The resolved mapping is saved to `.claude/risk-config.json` for subsequent runs.

You can pass a service name as a one-time override. This takes highest priority, skipping cache and auto-detection, and will not update `.claude/risk-config.json`:

```bash
/pagerduty:pre-commit-risk-scoring my-service-name
```

### Output

The command produces a structured risk assessment containing:

- **Active incidents** for the mapped service (highest-priority signal)
- **Incident history** summary over the last 90 days
- **Change analysis** with file-level and structural risk signals
- **Correlation findings** between current changes and past incidents
- **Risk score** from 0 (no risk) to 5 (critical), based on incident correlation and change analysis
- **Recommendations** based on identified risk factors

### Changing the mapped service

Delete `.claude/risk-config.json` and re-run `/pagerduty:pre-commit-risk-scoring` to pick a different service.

### Plugin structure

```text
.claude-plugin/
  marketplace.json            # Marketplace registry
  plugin.json                 # Plugin metadata
commands/
  pre-commit-risk-scoring.md  # Slash command definition
.mcp.json                     # PagerDuty MCP server declaration
```

### MCP servers

| Server    | Declared in                | Tools used                                                                             |
| --------- | -------------------------- | -------------------------------------------------------------------------------------- |
| PagerDuty | `.mcp.json` (plugin-local) | `get_service`, `list_services`, `list_incidents`, `list_incident_notes`, `list_service_change_events` |

## Plugin: Create PagerDuty Skill

**Early Access**: This feature requires enrollment in two Early Access programs:
- [PagerDuty Skills EA](https://www.pagerduty.com/early-access/)
- [PagerDuty Advance MCP/API EA](https://support.pagerduty.com/main/changelog/pagerduty-advance-mcpapi-support-is-now-in-early-access-for-advance-customers)

A plugin that provides an interactive workflow for creating and updating PagerDuty skills for AI agents. Skills allow you to encode your team's domain knowledge and operational procedures into structured instructions that the SRE Agent can follow during incident response.

### Usage

Launch the interactive skill builder:

```bash
/pagerduty:create-pagerduty-skill
```

The command guides you through:

1. **Mode selection**: Create new or update existing skill
2. **Scope selection**: Personal (user) or shared (account-level)
3. **Workflow definition**: What should the agent do?
4. **Skill configuration**: Name, description, instructions, examples, and metadata
5. **API deployment**: Immediate deployment to PagerDuty platform

### Features

- **Interactive interview**: Short, natural conversation extracts your domain knowledge
- **Two scope types**:
  - **Personal (user)**: Private skills visible only to you (max 25 per agent)
  - **Shared (account)**: Team-level skills visible to everyone (max 50 per agent)
- **Update mode**: Fetch existing skills and modify them (full replacement)
- **Validation**: Name format (kebab-case), length limits, token estimation
- **Integration references**: Automatically includes available SRE Agent integrations (Confluence, GitHub, Grafana, Datadog, etc.)
- **Immediate deployment**: Skills are available in the PagerDuty platform right after creation

### Skill structure

Skills consist of:
- **Name**: kebab-case identifier (max 60 chars, unique per agent/scope)
- **Description**: What the skill does (max 1024 chars)
- **Instructions**: Step-by-step execution guide (max 5000 tokens)
- **Examples**: Trigger conditions for automatic invocation
- **Metadata**: Version, author, team, type (experimental/production)

### MCP servers

| Server    | Declared in                | Tools used                                                                             |
| --------- | -------------------------- | -------------------------------------------------------------------------------------- |
| PagerDuty Advance MCP | `.mcp.json` (plugin-local) | `create_skill_tool`, `get_skill_tool`, `list_skills_tool`, `update_skill_tool` |

## Contributing

We welcome contributions! Please see [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines.

## License

This project is licensed under the Apache License 2.0 - see the [LICENSE](LICENSE) file for details.
