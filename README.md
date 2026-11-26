# PagerDuty Claude Code Plugins

[![License](https://img.shields.io/badge/License-Apache%202.0-blue.svg)](https://opensource.org/licenses/Apache-2.0)

PagerDuty's open source [Claude Code plugins](https://code.claude.com/docs/en/plugins) that bring operational intelligence into your development workflow.

## Before you start: one-time admin step

> **A PagerDuty account admin must install the Claude Code app once per PagerDuty account.** Until they do, nobody in the account can use this plugin.
>
> **[Install the Claude Code app in PagerDuty](https://app.pagerduty.com/oauth-apps/PU4FJQH)** (requires an account admin)

If the app is not installed, sign-in fails *after* you log in, with an error saying the app is not installed in the account. Send the link above to your PagerDuty admin, then try again.

With the admin step you do not need a client ID or client secret. See the [CIMD clients guide](https://docs.pagerduty.com/ai-automation/advance/pagerduty-mcp-server/cimd-clients) for details.

### Can't get the app installed?

Register your own OAuth app in PagerDuty and connect with its client ID and secret instead.

1. Register an OAuth app in PagerDuty with redirect URL `http://localhost:8080/callback`.
2. Add the server with your client ID and secret:

   ```bash
   MCP_CLIENT_SECRET=<client-secret> claude mcp add-json pagerduty \
     '{"type":"http","url":"https://mcp.pagerduty.com/mcp?tool_groups=full-tools,ai-agents","oauth":{"clientId":"<client-id>","callbackPort":8080}}' --client-secret
   ```

3. Run `/mcp`, select `pagerduty`, and complete the browser sign-in.

See [Use an OAuth client ID and client secret](https://docs.pagerduty.com/ai-automation/advance/pagerduty-mcp-server#use-an-oauth-client-id-and-client-secret). Never put the client secret in a file in your repository.

## Available plugins

| Plugin                                                    | Description                                                                                                  |
| --------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------ |
| pre-commit-risk-scoring | Pre-commit risk assessment using PagerDuty incident history, change event correlation, and git diff analysis  |
| investigate | Investigate a PagerDuty incident with the PagerDuty SRE Agent |
| on-call-handoff | Summarize incidents and on-call state for an on-call handoff |

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

### 3. Sign in to PagerDuty

The plugin bundles one MCP server, `pagerduty`, pointing at the PagerDuty-hosted [MCP server](https://docs.pagerduty.com/ai-automation/advance/pagerduty-mcp-server) (`https://mcp.pagerduty.com/mcp?tool_groups=full-tools,ai-agents`). Git commit history is read directly from the local repository.

Once your admin has completed the [one-time admin step](#before-you-start-one-time-admin-step), run `/mcp`, select `pagerduty`, and complete the browser sign-in.

## Troubleshooting

| Symptom | Fix |
| --- | --- |
| Sign-in fails after login with an error that the app is not installed in the account | A PagerDuty account admin must [install the Claude Code app](https://app.pagerduty.com/oauth-apps/PU4FJQH) once, then sign in again from `/mcp`. |
| `401` / unauthorized responses | Your session expired or was revoked. Run `/mcp`, select `pagerduty`, and re-authenticate. |
| `429` rate limit | The PagerDuty API is rate limiting requests. Wait a minute and retry. |

### Restricting scopes

The Claude Code app requests all scopes. If your organization wants narrower scopes, register your own Scoped OAuth app and connect with it as described in [Can't get the app installed?](#cant-get-the-app-installed).

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

To assess every commit on your current branch against its base branch, as a reviewer would see it:

```bash
/pagerduty:pre-commit-risk-scoring --branch
```

If there are no uncommitted changes and you're on a feature branch, the command switches to branch mode automatically. After an assessment of uncommitted changes, it offers to assess the whole branch too.

In branch mode, the base branch is detected from the open pull request (via the GitHub MCP server if you have it installed, otherwise the `gh` CLI), then the upstream tracking branch, then `main`, `master` or `develop`. If none of these work, the command asks you.

On first run, the plugin resolves your repository to a PagerDuty service through a fallback chain:

1. Explicit argument passed at invocation (one-time override, not cached)
2. Cached config in `.pagerduty/pagerduty-risk-config.json`
3. Backstage `catalog-info.yaml` annotation (`pagerduty.com/service-id`)
4. Auto-detection by matching the repository name against PagerDuty services
5. Manual input via interactive prompt

The resolved mapping is saved to `.pagerduty/pagerduty-risk-config.json` for subsequent runs.

If you used an earlier version of the plugin, a mapping saved in `.claude/risk-config.json` is copied to the new location on the next run. The old file is left in place, and you can delete it.

You can pass a service name as a one-time override. This takes highest priority, skipping cache and auto-detection, and will not update `.pagerduty/pagerduty-risk-config.json`. It can be combined with `--branch`:

```bash
/pagerduty:pre-commit-risk-scoring my-service-name
/pagerduty:pre-commit-risk-scoring --branch my-service-name
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

Delete `.pagerduty/pagerduty-risk-config.json` and re-run `/pagerduty:pre-commit-risk-scoring` to pick a different service.

### Plugin structure

```text
.claude-plugin/
  marketplace.json            # Marketplace registry
  plugin.json                 # Plugin metadata
commands/
  pre-commit-risk-scoring.md  # Slash command definitions
  investigate.md
  on-call-handoff.md
.mcp.json                     # PagerDuty MCP server declaration
```

### MCP servers

| Server    | Declared in                | Tools used                                                                             |
| --------- | -------------------------- | -------------------------------------------------------------------------------------- |
| PagerDuty | `.mcp.json` (plugin-local) | `browse_services` (`list`, `get`), `browse_incidents` (`list`, `list_notes`), `browse_change_events` (`list_service`) |
| GitHub (optional) | Your own Claude Code config | `list_pull_requests`, to find the base branch in branch mode. Falls back to the `gh` CLI and git if unavailable |

## Contributing

We welcome contributions! Please see [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines.

## License

This project is licensed under the Apache License 2.0 - see the [LICENSE](LICENSE) file for details.
