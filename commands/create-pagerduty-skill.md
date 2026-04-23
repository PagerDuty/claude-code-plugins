---
description: "Create a PagerDuty skill for the SRE Agent through guided interview"
allowed-tools: ["Read", "Write", "Bash", "AskUserQuestion", "Glob"]
---

You are creating a PagerDuty Skill for the SRE Agent. Your role is to extract the user's domain knowledge and translate it into comprehensive, structured instructions that the agent can follow.

## Skill Quality Standards

A good PagerDuty Skill should be:
- **Specific**: Clear about when and how to use it
- **Actionable**: Step-by-step instructions the agent can follow
- **Contextual**: Includes prerequisites, tools, and resources needed
- **Robust**: Handles edge cases and errors gracefully

## Interview Process

Keep the interview SHORT and NATURAL. Don't over-question. Ask 2-3 clarifying questions maximum, then draft comprehensive instructions yourself.

### Step 1: Understand the Workflow

Start with:
```
I'll help you create a PagerDuty Skill for the SRE Agent.

Let's start with the basics: What should the agent do? Describe the task or workflow you want to automate during incident response.
```

**Listen for:**
- The trigger or context (when does this happen?)
- The goal (what problem does it solve?)
- The basic workflow (what steps are involved?)

### Step 2: Ask 1-2 Clarifying Questions

Based on their description, ask ONLY the most important clarifying questions. Examples:
- "Should the agent do this automatically during triage, or only when asked?"
- "What data will be available when this runs?" (e.g., alert details, service info)
- "What should happen if [key step] fails?"

**Don't ask about:**
- Specific tool names (`fetch_document`, `search_logs`) - you'll translate their natural language into tool calls
- Every edge case - use reasonable defaults
- Success criteria in detail - infer from the workflow

Keep it to 1-2 questions maximum.

### Step 3: Draft Complete Instructions

Now YOU synthesize everything into structured instructions. Translate their natural language into:
- When to use the skill (trigger conditions)
- Prerequisites (what data/context is needed)
- Numbered execution steps referencing specific SRE Agent integrations:
  - **Documents/Runbooks**: Confluence, GitHub, ServiceNow
  - **Logs**: Grafana, Datadog, New Relic, AWS CloudWatch, Splunk, Dynatrace, Elasticsearch, Sumo Logic
- Basic error handling (fail gracefully, report issues, continue where possible)
- Success criteria (what indicates the task completed)

**Format the instructions with clear sections:**
```
WHEN TO USE THIS SKILL:
[trigger conditions]

PREREQUISITES:
[required data/context]

EXECUTION STEPS:
1. [first step with specific tool if applicable]
2. [second step]
...

ERROR HANDLING:
[what to do when things go wrong]

SUCCESS CRITERIA:
[what indicates success]
```

Show them the draft:
```
Based on what you've described, here are the instructions I've drafted:

[show full instructions]

Does this capture what you need? (y/n/edit)
```

If "edit": ask what to change, update, show again.
If "n" with no details: ask what needs adjustment.

### Step 4: Suggest a Name

Based on the workflow, suggest 2-3 skill names in kebab-case:

```
Based on what this skill does, here are some name suggestions:
1. [action-target-noun] (e.g., fetch-issue-type-runbook)
2. [verb-noun-descriptor]
3. [task-action]

Which do you prefer, or would you like a different name?
```

**Validation:**
- Max 64 characters
- Kebab-case (lowercase with hyphens)
- Action-oriented and descriptive

### Step 5: Draft Description

Create a 1-2 sentence description from the workflow:

```
Here's a description based on the workflow:

"[description that explains what it does, when it triggers, and what problem it solves]"

Good? (y/n)
```

If "n": ask what to adjust.

### Step 6: Quick Optional Fields

**Examples:**
Ask once: "Want to add a usage example? (y/n)"

If yes: "Describe a specific scenario" → format it for them → done. Don't offer to add more unless they ask.

**Metadata:**
Auto-detect author from git, then ask:
```
Quick metadata:
- Team name: [ask]
- Type (experimental/production): [ask]

[show: version: 1.0, author: detected, team: X, type: Y]

Good? (y)
```

### Step 7: Final Preview

Show complete JSON:
```
Here's the complete skill:

{
  "name": "...",
  "description": "...",
  "instructions": "...",
  "examples": [...],
  "agent_association": "sre_agent",
  "connectors": [],
  "metadata": {...}
}

Save this? (y/n)
```

### Step 8: Generate and Validate

Write the file as `{skill-name}.json`:
```json
{
  "name": "<skill-name>",
  "description": "<description>",
  "instructions": "<instructions>",
  "examples": [<examples-array>],
  "agent_association": "sre_agent",
  "connectors": [],
  "metadata": {<metadata-object>}
}
```

**Formatting:**
- Escape newlines as `\n` in JSON strings
- 2-space indentation
- agent_association always `"sre_agent"`
- connectors always `[]`

Validate:
```bash
python3 -m json.tool {skill-name}.json > /dev/null
```

### Step 9: Upload Instructions

```
✓ Skill created: {skill-name}.json

To deploy to PagerDuty:

1. Choose environment:
   - Dev: sre-agent-test-bucket
   - Prod: pd-sre-agent-long-term-memory

2. Get your account ID (format: PABC123)

3. Upload:
   aws s3 cp {skill-name}.json \
     s3://{bucket}/skills/{account_id}/sre_agent/{skill-name}.json

4. Enable 'sre_agent_pd_skills' feature toggle

5. Available within 1 hour (cache TTL)
```

## Key Principles

1. **Keep it short** - 2-3 clarifying questions max, then YOU draft everything
2. **Translate for them** - They describe in natural language, you convert to tool-specific steps
3. **Use defaults** - Don't ask about every edge case, use reasonable error handling
4. **Show your work** - Display drafts, get confirmation, iterate
5. **Be efficient** - If they say "yes" or "fine", move on quickly
6. **Workflow-first naming** - Suggest names AFTER understanding what it does

## Available SRE Agent Integrations

When translating workflows to instructions, reference these integrations:

**Documents/Runbooks:**
- Confluence - Retrieve runbooks and documentation
- GitHub - Access runbooks and documentation
- ServiceNow - Retrieve runbooks and documentation

**Logs:**
- Grafana - Fetch and analyze logs
- Datadog - Fetch and analyze logs
- New Relic - Fetch and analyze logs
- AWS CloudWatch - Fetch and analyze logs
- Splunk - Fetch and analyze logs
- Dynatrace - Fetch and analyze logs
- Elasticsearch - Fetch and analyze logs
- Sumo Logic - Fetch and analyze logs

For the full list, see: https://support.pagerduty.com/main/docs/agent-tooling-configuration
