---
description: "Generate an on-call incident summary or shift handoff using PagerDuty data"
argument-hint: "[team_name] [timeframe]"
allowed-tools: ["ToolSearch", "Read", "Write", "Bash", "AskUserQuestion", "mcp__plugin_pagerduty_pagerduty__browse_users", "mcp__plugin_pagerduty_pagerduty__browse_incidents", "mcp__plugin_pagerduty_pagerduty__browse_escalation_policies", "mcp__plugin_pagerduty_pagerduty__browse_schedules", "mcp__plugin_pagerduty_pagerduty__browse_services"]
---

Generate an on-call incident summary or shift handoff from PagerDuty data. Follow these steps in order. Do not skip steps. Do not silently degrade — if a required tool is unavailable, stop and tell the user.

Arguments: `$ARGUMENTS` — `[team_name] [timeframe]`. Both are optional.
- **team_name** — a non-numeric string with no time unit (e.g. `platform`, `payments`). Doubles as the config cache key.
- **timeframe** — a duration expression containing a number and unit (e.g. `7 days`, `2 weeks`, `30d`). Defaults to the escalation policy's `rotation_duration`, falling back to 7 days.

Parse `$ARGUMENTS` by scanning for a time-unit pattern first; treat any remaining tokens as `team_name`.

## Pre-flight

Before doing anything else, verify that the required MCP tools are available.

Call `ToolSearch` with query `"select:mcp__plugin_pagerduty_pagerduty__browse_incidents,mcp__plugin_pagerduty_pagerduty__browse_escalation_policies,mcp__plugin_pagerduty_pagerduty__browse_schedules,mcp__plugin_pagerduty_pagerduty__browse_services,mcp__plugin_pagerduty_pagerduty__browse_users"` to load the required tools.

If ToolSearch returns no results, STOP and tell the user:

```
PagerDuty MCP server is not available. This plugin requires it.

To fix:
1. Ask a PagerDuty account admin to install the Claude Code app once for your account (one-time step): https://app.pagerduty.com/oauth-apps/PU4FJQH
   If it is not installed, sign-in fails after login with an error that the app is not installed in the account.
   Can't get the app installed? Register your own OAuth app in PagerDuty and connect with its client ID and secret instead:
   - Register an OAuth app with redirect URL http://localhost:8080/callback
   - Run: MCP_CLIENT_SECRET=<client-secret> claude mcp add-json pagerduty '{"type":"http","url":"https://mcp.pagerduty.com/mcp?tool_groups=full-tools,ai-agents","oauth":{"clientId":"<client-id>","callbackPort":8080}}' --client-secret
   - Then sign in from /mcp
   Details: https://docs.pagerduty.com/ai-automation/advance/pagerduty-mcp-server#use-an-oauth-client-id-and-client-secret
2. Run /mcp, select "pagerduty", and complete the browser sign-in
3. Re-run /pagerduty:on-call-handoff
Docs: https://docs.pagerduty.com/ai-automation/advance/pagerduty-mcp-server/cimd-clients
```

Do NOT proceed without PagerDuty MCP.

## Step 1 — Scope resolution

Parse `$ARGUMENTS` to extract `team_name` and `timeframe`.

**Check cache:** Read `~/on-call-handoff/config.json`. Look for a profile key matching `team_name` (if provided).

- If a matching profile is found and contains non-empty `escalation_policies` and `services`, use it. Skip to the "Confirm scope" block below.
- Otherwise: resolve fresh from PagerDuty.

**Resolve from PagerDuty (run in parallel):**
1. Call `mcp__plugin_pagerduty_pagerduty__browse_users` with action `get` to get the current user's email and teams.
2. Call `mcp__plugin_pagerduty_pagerduty__browse_escalation_policies` with action `list` to list policies. If `team_name` was provided, pass it as `query` to filter by name match. For each matching policy, call `mcp__plugin_pagerduty_pagerduty__browse_escalation_policies` with action `get` and `policy_id` to get its schedules; for each schedule call `mcp__plugin_pagerduty_pagerduty__browse_schedules` with action `get` and `schedule_id` to retrieve `rotation_duration`.
3. Call `mcp__plugin_pagerduty_pagerduty__browse_services` with action `list` and `team_ids` set to the current user's team IDs from step 1.

**Profile key:** use `team_name` if provided, otherwise the user's email.

**Timeframe:** use `timeframe` from `$ARGUMENTS` if provided. Otherwise use the first escalation policy's `rotation_duration`; fall back to 7 days if unavailable.

**Save cache:** after resolving, write the profile to `~/on-call-handoff/config.json`, merging with any existing profiles:
```json
{
  "[profile-key]": {
    "escalation_policies": [{"id": "...", "name": "...", "rotation_duration": "..."}],
    "services": [{"id": "...", "name": "..."}]
  }
}
```
If the write fails, note the error and continue — do not block the report.

**Confirm scope:** use `AskUserQuestion` to confirm the selected team/escalation policy and timeframe before proceeding.

<HARD-GATE>Do not proceed to Step 2 until the user has confirmed scope and timeframe.</HARD-GATE>

## Step 2 — Data gathering

Run all three calls in parallel:

1. **Active incidents** — call `mcp__plugin_pagerduty_pagerduty__browse_incidents` with action `list`, `statuses: ["triggered", "acknowledged"]`, and `service_ids` set to the services in the resolved profile.

2. **All incidents in window** — call `mcp__plugin_pagerduty_pagerduty__browse_incidents` with action `list` and `since`/`until` covering the timeframe, for the same `service_ids`. Omit `statuses` to include all statuses. If the response has `truncated: true`, call again with `offset` advanced by `limit` until it does not.

3. **Stale check** — from call 1, flag any triggered incident open >7 days.

From the collected data derive:
- **Top services by incident volume** (from call 2)
- **Recurring alert patterns** — deduplicate incidents by title; a title appearing ≥3× in the window is a recurring pattern
- **Volume signal** — flag if total incident count for the window is >50 for a 7-day timeframe (or proportionally adjusted)
- **Cap note** — stop paging call 2 after 1000 incidents; if you stop there, the history is partial; note the oldest incident's timestamp

## Step 3 — Output

Adapt output style to timeframe: ≤14 days → handoff-focused (Active first); >14 days → trend-focused (Incident Trends first).

```
## On-Call Summary — [Escalation Policy Name]
Period: [start] → [end] | Generated: [date]

### Active  [omit if no active incidents]
[triggered/acknowledged incidents — one line each: ID, service, age, assignee]

### Incident Trends
Top services: [service — N incidents] (one line each, top 5 max)
Recurring alerts: [title — N occurrences] (or "None")
Volume: [N incidents in period; note if unusually high]

### Watch List
[stale incidents (open >7d), services with sharp volume spikes, recurring alert patterns needing attention]

### Recommended Actions
[3–5 concrete items, most urgent first]
```

Keep it scannable — one line per item. Omit any section that has nothing to report.

## Step 4 — Follow-up

Use `AskUserQuestion`: "Found [N] incidents in [timeframe]. What would you like to do next?"

- **Deep analysis with SRE Agent** — use a second `AskUserQuestion` to let the user pick an incident from the report. Dynamically build the options from the data already collected:
  - Up to 3 options: prioritize triggered/acknowledged incidents first (label = `#[number] [title truncated to 50 chars]`, description = service name + age); fill remaining slots with notable watch list items if fewer than 3 active incidents
  - 4th option always: **"List all incidents"** — output every incident from the handoff window as a numbered list (one per line: `#[number] [title] — [service] — [age]`), then use `AskUserQuestion` with a single "Other" option so the user can type the incident number or alphanumeric ID they want to investigate
  Once the user selects or types an ID, tell them:
  > Run `/pagerduty:investigate <incident-id>` to open a multi-turn SRE Agent investigation. It maintains session context across follow-up questions — root cause, deployment correlation, similar past incidents, and more.
- **Save to file** — write the summary to `./on-call-reports/[escalation-policy-slug]_[YYYY-MM-DD_HH-MM].md`. Create `./on-call-reports/` if it does not exist. Confirm the path to the user.
- **Done** — finish without saving.
