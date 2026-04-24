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

- **agent_type**: Always `"sre"` (only SRE Agent supports skills currently)
- **scope**: REQUIRED - `"account"` (shared, team-level) or `"user"` (personal, individual)
- **name**: kebab-case, max 60 chars, unique per (agent, scope) - same name can exist in both scopes
- **description**: max 1024 chars
- **instructions**: max 5000 tokens total (entire skill document)
- **limits**: 
  - Account scope: Max 50 skills per agent
  - User scope: Max 25 skills per agent
- **immutable fields**: scope and name cannot be changed after creation (must delete and recreate)

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

**Your role**: You are the translator between the user's natural language and agent-optimized skill definitions. The user describes what they want; YOU synthesize it into structured, agent-friendly instructions, descriptions, and examples.

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

Store the mode as `create` or `update` and proceed to Step 2.

### Step 2: Scope Selection

Ask the user to choose the scope:

```
Is this a personal skill or a shared team skill?
1. Personal (visible only to you)
2. Shared (visible to everyone in your PagerDuty account)

(1/2)
```

Map response to scope:
- 1 → `"user"`
- 2 → `"account"`

**Important**: The same skill name can exist in both scopes independently. Identity is `(agent, scope, name)`.

### Step 3: Skill Selection (UPDATE mode only)

**For UPDATE mode, follow these steps:**

**3a. List Existing Skills:**

Call `list_skills_tool` with agent_type="sre" and the chosen scope.

If no skills exist:
```
No {scope} skills found. Let's create one instead.
```
Switch to CREATE mode and continue to Step 4.

If skills exist, display them:
```
Existing skills:

1. fetch-issue-type-runbook
   Description: Use this skill to fetch issue specific runbooks
   Metadata: version=1.0, author=user@example.com, team=platform

2. diagnose-database-slowness
   Description: Automated diagnosis of database performance issues
   Metadata: version=2.1, author=user@example.com, team=infrastructure

Which skill would you like to update? (enter number or name)
```

**3b. Fetch Current Skill:**

Use `get_skill_tool` with agent_type="sre", scope, and skill_name to retrieve the full skill document.

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

### Step 4: Check Skill Limit (CREATE mode only)

**For CREATE mode:**

Call `list_skills_tool` with agent_type="sre" and the chosen scope. Count results and check against limit:
- Account scope: limit is 50
- User scope: limit is 25

If limit reached:
```
⚠️ Limit reached: You already have {count} {scope} skills (the maximum for {scope} scope).

Existing skills:
1. skill-name-1
2. skill-name-2
...

You can update an existing skill or delete one first.
Would you like to switch to update mode? (y/n)
```

If yes, switch to UPDATE mode and go to Step 3a.
If no, exit with instructions to delete a skill first.

### Step 5: Understand the Workflow

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

### Step 6: Ask 1-2 Clarifying Questions

Based on their description, ask ONLY the most important clarifying questions. Examples:
- "Should the agent do this automatically during triage, or only when asked?"
- "What data will be available when this runs?" (e.g., alert details, service info)
- "What should happen if [key step] fails?"

**Don't ask about:**
- Specific tool names - you'll translate their natural language into integration references
- Every edge case - use reasonable defaults
- Success criteria in detail - infer from the workflow

Keep it to 1-2 questions maximum.

### Step 7: Draft Complete Instructions

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

### Step 8: Suggest a Name (CREATE mode) or Show Name (UPDATE mode)

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
- Check uniqueness: call `list_skills_tool` with the chosen scope and verify name doesn't exist
- If name exists in that scope, show error and ask for different name
- Note: Same name can exist in both scopes, but must be unique within the chosen scope

**For UPDATE mode:**

```
Current name: {current_name}

Note: Skill names cannot be changed after creation. The name will remain "{current_name}".
```

### Step 9: Draft Description

YOU create a 1-2 sentence description from the workflow. Don't ask the user to write it - interpret their workflow and draft an agent-optimized description yourself.

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
- Written for the agent to understand, not just human-readable
- Focus on trigger conditions and outcomes

If "n": ask what to adjust and revise.

### Step 10: Quick Optional Fields

**Examples (trigger conditions/prompts):**

```
{if UPDATE and examples: Current examples: {len(examples)} example(s)
{show current examples}}

Want to {add/update/keep} examples of when to invoke this skill? (y/n{/keep if UPDATE})
```

If yes: 
```
When should the agent invoke this skill? Describe the conditions or situations.
```

Listen to their response, then YOU translate it into 2-3 clear, agent-friendly trigger conditions. Format them as:
- Specific conditions in alert data: "Invoke when alert custom details contain {key}: {value}"
- User prompts: "Use when the user asks to {action}"  
- Automatic triggers: "Trigger during {phase} if {condition}"

Show your draft and confirm:
```
Based on that, here are the trigger examples:

1. {example_1}
2. {example_2}
3. {example_3}

Good? (y/n)
```

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

### Step 11: Final Preview

Display complete skill structure:

```
{if CREATE: Here's the complete skill:}
{if UPDATE: Here are your changes:}

Scope: {scope} ({if account: "shared with your team" | if user: "personal to you"})
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
  "agent_type": "sre",
  "scope": "{scope}",
  "name": "{name}",
  "description": "{description}",
  "instructions": "{instructions}",
  "examples": {examples or null},
  "connectors": null,
  "metadata": {metadata}
}
```

Then re-ask: "Ready to proceed? (y/n)"

### Step 12: API Execution

**For CREATE mode:**

```
Creating skill via PagerDuty API...
```

Call `mcp__pagerduty-advance-mcp-__create_skill_tool` with parameters:
- `agent_type`: "sre"
- `scope`: "account" or "user" (from Step 2)
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
- `agent_type`: "sre"
- `scope`: "account" or "user" (from Step 2)
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
   ❌ Error: Skill "{name}" already exists in {scope} scope.
   
   Choose a different name or switch to update mode.
   Would you like to update the existing skill instead? (y/n)
   ```

2. **Skill not found (UPDATE only):**
   ```
   ❌ Error: Skill "{name}" not found in {scope} scope.
   
   The skill may have been deleted. Would you like to create it instead? (y/n)
   ```

3. **Skill limit reached (CREATE only):**
   ```
   ❌ Error: Maximum of {50 for account | 25 for user} {scope} skills reached.
   
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

### Step 13: Success

On successful API call:

```
✅ Skill {created/updated}: {skill-name}

Scope: {scope} ({if account: "shared with your team" | if user: "personal to you"})
Your skill is now available to the SRE Agent!

Next steps:
1. The skill is immediately available in the PagerDuty platform
2. Test the skill in an incident or via the SRE Agent interface
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
  "agent_type": "sre",
  "scope": "{scope}",
  "connectors": [],
  "metadata": {metadata}
}
```

## Key Principles

1. **Keep it short** - 2-3 clarifying questions max, then YOU draft everything
2. **Translate everything** - They describe in natural language, you convert ALL fields (description, instructions, examples) into agent-optimized format
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
