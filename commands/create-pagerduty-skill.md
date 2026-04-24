---
description: "Create or update PagerDuty skills for AI agents through guided interview"
allowed-tools: [
  "ToolSearch",
  "Read", 
  "Write",
  "Bash", 
  "AskUserQuestion", 
  "Glob",
  "mcp__pagerduty-advance-mcp-__create_skill_tool",
  "mcp__pagerduty-advance-mcp-__get_skill_tool",
  "mcp__pagerduty-advance-mcp-__list_skills_tool",
  "mcp__pagerduty-advance-mcp-__update_skill_tool"
]
---

You are creating or updating a PagerDuty Skill for AI agents. Your role is to extract the user's domain knowledge and translate it into comprehensive, structured instructions that the agent can follow.

## Pre-flight Checks

**CRITICAL: Before proceeding, verify PagerDuty Advance MCP is available.**

1. Use ToolSearch to check for `mcp__pagerduty-advance-mcp-__create_skill_tool`
2. If not found, display error and EXIT (do not proceed):
   ```
   ❌ PagerDuty Advance MCP is not available.
   
   This command requires the PagerDuty Advance MCP server to create and manage skills via API.
   
   To fix:
   1. Ensure PagerDuty Advance MCP is configured in .mcp.json
   2. Set your PAGERDUTY_API_KEY in the environment (.claude/settings.local.json or ~/.claude/settings.json)
   3. Restart Claude Code
   4. Re-run /pagerduty:create-pagerduty-skill
   
   For setup help, see: https://github.com/PagerDuty/pagerduty-advance-mcp
   ```
3. Do NOT proceed without MCP access - no silent degradation

## API Constraints

Skills are managed via the PagerDuty Advance MCP API with these constraints:

- **agent_type**: Must be `"sre"`, `"insights"`, or `"shift"` (NOT `"sre_agent"`)
- **name**: kebab-case, max 60 chars, unique per (account, agent_type)
- **description**: max 1024 chars
- **instructions**: max 5000 tokens total (entire skill document)
- **limit**: Max 5 skills per (account, agent_type) pair

**Validation rules:**
- Name format: `^[a-z0-9]+(-[a-z0-9]+)*$` (kebab-case only)
- Token estimation: `chars / 4 ≈ tokens`
- Warn at 90% of 5000 token limit (4500 tokens)

## Skill Quality Standards

A good PagerDuty Skill should be:
- **Specific**: Clear about when and how to use it
- **Actionable**: Step-by-step instructions the agent can follow
- **Contextual**: Includes prerequisites, tools, and resources needed
- **Robust**: Handles edge cases and errors gracefully

## Interview Process

Keep the interview SHORT and NATURAL. Don't over-question. Ask 2-3 clarifying questions maximum, then draft comprehensive instructions yourself.

### Step 1: Mode Selection

First, determine whether to create a new skill or update an existing one.

Ask:
```
I'll help you create or update a PagerDuty Skill.

What would you like to do?
1. Create a new skill
2. Update an existing skill

(1/2)
```

Store the mode as `create` or `update` and proceed to the appropriate workflow.

### Step 2: Skill Selection (UPDATE mode only)

**For UPDATE mode, follow these steps:**

**2a. Choose Agent Type:**
```
Which agent type is this skill for?
1. SRE Agent (incident response & diagnosis)
2. Insights Agent (analytics & reporting)
3. Shift Agent (on-call & scheduling)

(1/2/3)
```

Map response to agent_type:
- 1 → `"sre"`
- 2 → `"insights"`
- 3 → `"shift"`

**2b. List Existing Skills:**

Call `list_skills_tool` with the chosen agent_type.

If no skills exist:
```
No skills found for {agent_type} agent. Let's create one instead.
```
Switch to CREATE mode and continue to Step 3.

If skills exist, display them:
```
Existing skills for {agent_type} agent:

1. fetch-issue-type-runbook
   Description: Use this skill to fetch issue specific runbooks
   Metadata: version=1.0, author=mmayp@pagerduty, team=signal

2. diagnose-database-slowness
   Description: Automated diagnosis of database performance issues
   Metadata: version=2.1, author=db-team@pagerduty, team=platform

Which skill would you like to update? (enter number or name)
```

**2c. Fetch Current Skill:**

Use `get_skill_tool` with agent_type and skill_name to retrieve the full skill document.

Display current configuration:
```
Current skill configuration:

Name: {name}
Description: {description}

Instructions: [showing first 300 chars]
{instructions[:300]}...

Examples: {len(examples) if examples else 0} example(s)
Metadata: version={version}, author={author}, team={team}, type={type}

Ready to update this skill? (y/n)
```

If "n": ask if they want to pick a different skill or exit.

Store all current values for use in interview prompts.

### Step 3: Agent Type Selection (CREATE mode only)

**For CREATE mode:**

Same as Step 2a, but after selection check skill limit:

Call `list_skills_tool` and count results. If count >= 5:
```
⚠️ Limit reached: {agent_type} agent already has 5 skills (the maximum).

Existing skills:
1. skill-name-1
2. skill-name-2
...

You can update an existing skill or delete one first.
Would you like to switch to update mode? (y/n)
```

If yes, switch to UPDATE mode and go to Step 2b.
If no, exit with instructions to delete a skill first.

### Step 4: Understand the Workflow

Start with:
```
{if CREATE: Let's create your skill.}
{if UPDATE: Let's update your skill.}

{if UPDATE: Current description: "{current_description}"}

What should the agent do? Describe the task or workflow you want to automate during incident response.

{if UPDATE: Press Enter to keep the current workflow, or describe changes/new workflow}
```

**Listen for:**
- The trigger or context (when does this happen?)
- The goal (what problem does it solve?)
- The basic workflow (what steps are involved?)

### Step 5: Ask 1-2 Clarifying Questions

Based on their description, ask ONLY the most important clarifying questions. Examples:
- "Should the agent do this automatically during triage, or only when asked?"
- "What data will be available when this runs?" (e.g., alert details, service info)
- "What should happen if [key step] fails?"

**Don't ask about:**
- Specific tool names - you'll translate their natural language into integration references
- Every edge case - use reasonable defaults
- Success criteria in detail - infer from the workflow

Keep it to 1-2 questions maximum.

### Step 6: Draft Complete Instructions

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
{if CREATE: Based on what you've described, here are the instructions I've drafted:}
{if UPDATE: Here are the updated instructions:}

[show full instructions]

Does this capture what you need? (y/n/edit)
```

If "edit": ask what to change, update, show again.
If "n" with no details: ask what needs adjustment.

**Token count check:**
After finalizing instructions, estimate token count: `len(instructions) / 4`
If > 4500 tokens (90% of 5000 limit), warn:
```
⚠️ Instructions are ~{estimated_tokens} tokens (90%+ of 5000 limit).
Consider condensing to avoid hitting the API limit.
```

### Step 7: Suggest a Name (CREATE mode) or Show Name (UPDATE mode)

**For CREATE mode:**

Based on the workflow, suggest 2-3 skill names in kebab-case:

```
Based on what this skill does, here are some name suggestions:
1. [action-target-noun] (e.g., fetch-issue-type-runbook)
2. [verb-noun-descriptor]
3. [task-action]

Which do you prefer, or would you like a different name?
```

**Validation:**
- Max 60 characters (not 64)
- Kebab-case format: `^[a-z0-9]+(-[a-z0-9]+)*$`
- Action-oriented and descriptive
- Check uniqueness: call `list_skills_tool` and verify name doesn't exist
- If name exists, show error and ask for different name

**For UPDATE mode:**

```
Current name: {current_name}

Note: Skill names cannot be changed after creation. The name will remain "{current_name}".
```

### Step 8: Draft Description

Create a 1-2 sentence description from the workflow:

```
{if UPDATE: Current description: "{current_description}"}

Here's a description based on the workflow:

"{generated_description}"

Good? (y/n)
{if UPDATE: Press Enter to keep current}
```

**Validation:**
- Max 1024 characters
- Should explain what it does, when it triggers, and what problem it solves

If "n": ask what to adjust.

### Step 9: Quick Optional Fields

**Examples (trigger conditions/prompts):**

```
{if UPDATE and examples: Current examples: {len(examples)} example(s)
{show current examples}}

Want to {add/update/keep} examples of when to invoke this skill? (y/n{/keep if UPDATE})
```

If yes: 
```
What conditions or user prompts should trigger this skill? For example:
- 'Invoke this skill when alert custom details contain an issue_runbook key'
- 'Use this when the user asks to fetch a runbook'
- 'Trigger during initial triage if X condition is present'

Provide 2-3 examples (one per line):
```

Take their input and format as array of strings. Keep it short - 2-3 examples maximum.

If "keep" (UPDATE mode): preserve current examples.

**Metadata:**

Auto-detect author from git:
```bash
git config user.email
```

Then ask:
```
{if UPDATE: Current metadata: version={version}, author={author}, team={team}, type={type}}

Metadata:
- Author: {detected_email} {if UPDATE: (press Enter to keep current: {author})}
- Team name: {if UPDATE: [current: {team}]} [ask]
- Type (experimental/production): {if UPDATE: [current: {type}]} [ask]
- Version: {if UPDATE: {suggest_increment(current_version)} | if CREATE: 1.0}

Confirm these values? (y/n)
```

**Version increment logic (UPDATE mode):**
- Parse current version (e.g., "1.0", "1.9", "2.3")
- Suggest minor increment: 1.0 → 1.1, 1.9 → 1.10, 2.3 → 2.4
- Allow user to override with custom version

### Step 10: Final Preview

Display complete skill structure:

```
{if CREATE: Here's the complete skill:}
{if UPDATE: Here are your changes:}

Agent Type: {agent_type}
Name: {name}
Description: {description}

Instructions: [{char_count} chars, ~{estimated_tokens} tokens]
{instructions}

Examples: {examples or "none"}

Connectors: [] (empty)

Metadata:
  version: {version}
  author: {author}
  team: {team}
  type: {type}

{if CREATE: Create this skill?}
{if UPDATE: Update this skill? (Note: This is a full replacement - all fields will be updated)}
(y/n/preview-json)
```

**Option: preview-json**

If user types "preview-json", show the exact API payload:
```json
{
  "agent_type": "{agent_type}",
  "name": "{name}",
  "description": "{description}",
  "instructions": "{instructions}",
  "examples": {examples or null},
  "connectors": null,
  "metadata": {metadata}
}
```

Then re-ask: "Ready to proceed? (y/n)"

### Step 11: API Execution

**For CREATE mode:**

```
Creating skill via PagerDuty API...
```

Call `mcp__pagerduty-advance-mcp-__create_skill_tool` with parameters:
- `agent_type`: "sre" | "insights" | "shift"
- `name`: kebab-case string
- `description`: string
- `instructions`: string  
- `examples`: array of strings or null (if empty, pass null not [])
- `connectors`: null (always)
- `metadata`: object with {version, author, team, type}

**For UPDATE mode:**

```
Updating skill via PagerDuty API...
Note: This is a full replacement. All fields will be updated.
```

Call `mcp__pagerduty-advance-mcp-__update_skill_tool` with parameters:
- `agent_type`: "sre" | "insights" | "shift"
- `skill_name`: immutable skill name (required)
- `description`: string
- `instructions`: string
- `examples`: array of strings or null
- `connectors`: null (always)
- `metadata`: object with {version, author, team, type}

**Error Handling:**

Handle these common API errors:

1. **Name collision (CREATE only):**
   ```
   ❌ Error: Skill "{name}" already exists for {agent_type} agent.
   
   Choose a different name or switch to update mode.
   Would you like to update the existing skill instead? (y/n)
   ```

2. **Skill not found (UPDATE only):**
   ```
   ❌ Error: Skill "{name}" not found.
   
   The skill may have been deleted. Would you like to create it instead? (y/n)
   ```

3. **Skill limit reached (CREATE only):**
   ```
   ❌ Error: Maximum of 5 skills per agent reached for {agent_type}.
   
   Existing skills: [list from list_skills_tool]
   
   You can update an existing skill or delete one first.
   Would you like to switch to update mode? (y/n)
   ```

4. **Validation errors (name format, length, tokens):**
   ```
   ❌ Error: {error_message}
   
   {specific fix guidance based on error}
   ```
   Return to the specific field that failed and ask user to correct it.

5. **Authentication/API errors:**
   ```
   ❌ API Error: {error_message}
   
   Possible causes:
   - PAGERDUTY_API_KEY not set or invalid
   - Network connectivity issues
   - PagerDuty service outage
   
   Check your API key and try again.
   ```

### Step 12: Success

On successful API call:

```
✅ Skill {created/updated}: {skill-name}

Your skill is now available to the {agent_type} agent!

Next steps:
1. The skill is immediately available in the PagerDuty platform
2. Test the skill in an incident or via the {agent_type} agent interface
3. Monitor skill usage and iterate as needed

{if CREATE: 
Optional: Would you like to save a local JSON backup for your records? (y/n)
}

{if UPDATE:
Changes applied:
{summarize what changed: description, instructions, examples, metadata}
}
```

**Optional JSON backup (CREATE mode only):**

If user says yes, write `{skill-name}.json` in current directory:
```json
{
  "name": "{name}",
  "description": "{description}",
  "instructions": "{instructions}",
  "examples": {examples or []},
  "agent_type": "{agent_type}",
  "connectors": [],
  "metadata": {metadata}
}
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
