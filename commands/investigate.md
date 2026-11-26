---
description: "Multi-turn SRE Agent investigation for a PagerDuty incident"
argument-hint: "[incident-id]"
allowed-tools: ["ToolSearch", "AskUserQuestion", "Write", "mcp__plugin_pagerduty_pagerduty__browse_incidents", "mcp__plugin_pagerduty_pagerduty__paige_sre_agent_send_message", "mcp__plugin_pagerduty_pagerduty__paige_sre_agent_get_response"]
---

Investigate a PagerDuty incident with the SRE Agent in a multi-turn conversation. Follow these steps in order. Do not silently degrade — if a required tool is unavailable, stop and tell the user.

Arguments: `$ARGUMENTS` — `[incident-id]`. Optional.
- Alphanumeric ID (e.g. `Q3AHWAR33KUK3G`) — used directly.
- Numeric or `#`-prefixed (e.g. `#971097`) — resolve to alphanumeric via `browse_incidents` (action `list`).
- Absent — list active/triggered incidents and let the user pick.

## Pre-flight

Call `ToolSearch` with query `"select:mcp__plugin_pagerduty_pagerduty__paige_sre_agent_send_message,mcp__plugin_pagerduty_pagerduty__paige_sre_agent_get_response,mcp__plugin_pagerduty_pagerduty__browse_incidents"`.

If all tools are missing, STOP:

```
PagerDuty MCP is not available. This command requires it.

To fix:
1. Ask a PagerDuty account admin to install the Claude Code app once for your account (one-time step): https://app.pagerduty.com/oauth-apps/PU4FJQH
   If it is not installed, sign-in fails after login with an error that the app is not installed in the account.
   Can't get the app installed? Register your own OAuth app in PagerDuty and connect with its client ID and secret instead:
   - Register an OAuth app with redirect URL http://localhost:8080/callback
   - Run: MCP_CLIENT_SECRET=<client-secret> claude mcp add-json pagerduty '{"type":"http","url":"https://mcp.pagerduty.com/mcp?tool_groups=full-tools,ai-agents","oauth":{"clientId":"<client-id>","callbackPort":8080}}' --client-secret
   - Then sign in from /mcp
   Details: https://docs.pagerduty.com/ai-automation/advance/pagerduty-mcp-server#use-an-oauth-client-id-and-client-secret
2. Run /mcp, select "pagerduty", and complete the browser sign-in
3. Re-run /pagerduty:investigate
Docs: https://docs.pagerduty.com/ai-automation/advance/pagerduty-mcp-server/cimd-clients
```

If `browse_incidents` is present but `paige_sre_agent_send_message` or `paige_sre_agent_get_response` is missing, STOP:

```
The PagerDuty SRE Agent tool is not available. It requires PagerDuty Advance on your account
and the ai-agents tool group on the PagerDuty MCP server.

See https://docs.pagerduty.com/ai-automation/advance/pagerduty-mcp-server#step-1-choose-an-authorization-method
```

## Step 1 — Incident resolution

Parse `$ARGUMENTS`:

**Alphanumeric ID** (contains letters, e.g. `Q3AHWAR33KUK3G`): use directly as `incident_id`. Skip to Step 2.

**Numeric or `#`-prefixed** (e.g. `#971097` or `971097`): strip `#`, call `mcp__plugin_pagerduty_pagerduty__browse_incidents` with action `list` and find the incident whose `incident_number` matches. Use that incident's `id` field as `incident_id`. If no match found, tell the user and ask them to provide the alphanumeric ID directly.

**No argument**: call `mcp__plugin_pagerduty_pagerduty__browse_incidents` with action `list`, `statuses: ["triggered", "acknowledged"]`, and `request_scope: "assigned"`. Present up to 10 results via `AskUserQuestion` (one line each: ID, title truncated to 60 chars, service, age). If no active incidents, ask: enter an ID manually, or search the last 24 hours of resolved incidents.

<HARD-GATE>Do not call the SRE Agent until an incident ID is confirmed.</HARD-GATE>

## Step 2 — Initial analysis

The SRE Agent is a two-step interaction: `paige_sre_agent_send_message` submits a message and returns a `conversation_id` immediately; `paige_sre_agent_get_response` returns the answer.

**Send.** Call `mcp__plugin_pagerduty_pagerduty__paige_sre_agent_send_message` with:
- `incident_id`: the resolved alphanumeric ID
- `message`: `"Summarize incident [id]: what happened, root cause, and top recommended actions."`

**AI Actions cost gate.** The SRE Agent consumes AI Actions (formerly PagerDuty Advance credits). On the first call in a conversation, do **not** pass `user_confirmed_ai_actions_cost`. The response has status `confirmation_required` and a `warning`. Show that warning to the user via `AskUserQuestion` (options: "Yes, continue" / "No, stop"). Only if they choose yes, repeat the same call with `user_confirmed_ai_actions_cost: "yes"`, and pass it on every later send in this conversation without asking again. If they decline, stop and do not call the SRE Agent again.

Store the `conversation_id` from the response. Every follow-up must use this exact value.

**Poll.** Call `mcp__plugin_pagerduty_pagerduty__paige_sre_agent_get_response` with the `conversation_id`:
- `is_complete: false` (status `working`) is expected. Call it again with the same `conversation_id` until `is_complete` is true.
- `is_complete: true` and status `idle`: the `messages` field holds the answer. Display it.
- `is_complete: true` and status `no_response`: the run was interrupted. Resend the same message once with `paige_sre_agent_send_message`, then poll again. If it fails again, tell the user the SRE Agent is unavailable right now and stop.

If a send returns status `busy`, a previous message is still being answered: poll with `paige_sre_agent_get_response` instead of resending.

## Step 3 — Conversation loop

After each SRE Agent response, call `AskUserQuestion`: "What would you like to explore next?"

- **Ask a follow-up question** — user types their question via "Other" input. Call `mcp__plugin_pagerduty_pagerduty__paige_sre_agent_send_message` with `incident_id`, `message=<user question>`, and `user_confirmed_ai_actions_cost: "yes"`, then poll with `paige_sre_agent_get_response` and the stored `conversation_id` as in Step 2. Display the result. Return to the top of this step.
- **Save conversation** — write the full conversation transcript (all questions and SRE Agent responses, in order) to `./on-call-reports/sre-[incident-id]_[YYYY-MM-DD_HH-MM].md`. Create `./on-call-reports/` if it does not exist. Confirm the path to the user. Return to the top of this step — the user may continue after saving.
- **Done** — exit.

Never resend a message while its response is still `working`.
