<purpose>
Interactive configuration of GSD workflow agents (research, plan_check, verifier) and model profile selection via multi-question prompt. Updates .planning/config.json with user preferences.
</purpose>

<required_reading>
Read all files referenced by the invoking prompt's execution_context before starting.
</required_reading>

<process>

<step name="ensure_and_load_config">
Ensure config exists and load current state:

```bash
node ~/.claude/get-shit-done/bin/gsd-tools.js config-ensure-section
INIT=$(node ~/.claude/get-shit-done/bin/gsd-tools.js state load)
```

Creates `.planning/config.json` with defaults if missing and loads current config values.
</step>

<step name="read_current">
```bash
cat .planning/config.json
```

Parse current values (default to `true` if not present):
- `workflow.research` — spawn researcher during plan-phase
- `workflow.plan_check` — spawn plan checker during plan-phase
- `workflow.verifier` — spawn verifier during execute-phase
- `model_profile` — which model each agent uses (default: `balanced`)
- `git.branching_strategy` — branching approach (default: `"none"`)
</step>

<step name="present_settings">
Use AskUserQuestion with current values pre-selected:

```
AskUserQuestion([
  {
    question: "Which model profile for agents?",
    header: "Model",
    multiSelect: false,
    options: [
      { label: "Quality", description: "Opus everywhere except verification (highest cost)" },
      { label: "Balanced (Recommended)", description: "Opus for planning, Sonnet for execution/verification" },
      { label: "Budget", description: "Sonnet for writing, Haiku for research/verification (lowest cost)" }
    ]
  },
  {
    question: "Spawn Plan Researcher? (researches domain before planning)",
    header: "Research",
    multiSelect: false,
    options: [
      { label: "Yes", description: "Research phase goals before planning" },
      { label: "No", description: "Skip research, plan directly" }
    ]
  },
  {
    question: "Spawn Plan Checker? (verifies plans before execution)",
    header: "Plan Check",
    multiSelect: false,
    options: [
      { label: "Yes", description: "Verify plans meet phase goals" },
      { label: "No", description: "Skip plan verification" }
    ]
  },
  {
    question: "Spawn Execution Verifier? (verifies phase completion)",
    header: "Verifier",
    multiSelect: false,
    options: [
      { label: "Yes", description: "Verify must-haves after execution" },
      { label: "No", description: "Skip post-execution verification" }
    ]
  },
  {
    question: "Git branching strategy?",
    header: "Branching",
    multiSelect: false,
    options: [
      { label: "None (Recommended)", description: "Commit directly to current branch" },
      { label: "Per Phase", description: "Create branch for each phase (gsd/phase-{N}-{name})" },
      { label: "Per Milestone", description: "Create branch for entire milestone (gsd/{version}-{name})" }
    ]
  }
])
```
</step>

<step name="agent_teams_settings">
**Only show this step if `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1` is detected in the environment.**

If the env var is NOT set, skip this step entirely.

```
AskUserQuestion([
  {
    question: "Use Agent Teams for parallel workflows? (experimental)",
    header: "Agent Teams",
    multiSelect: false,
    options: [
      { label: "No (Recommended)", description: "Use standard Task() flow — stable, proven" },
      { label: "Yes", description: "Use Agent Teams — teammates with messaging, self-claiming tasks" }
    ]
  }
])
```

**If "Yes":**

```
AskUserQuestion([
  {
    question: "Which workflows should use Agent Teams?",
    header: "Teams Scope",
    multiSelect: true,
    options: [
      { label: "Research", description: "Researchers challenge each other's findings via messaging" },
      { label: "Planning", description: "Planner + checker iterate via direct messaging (no lead involvement)" },
      { label: "Execution", description: "Self-claiming executors — no wave idle time (most impactful)" }
    ]
  }
])
```

Map selections to config:
- "Research" selected → `use_for_research: true`
- "Planning" selected → `use_for_planning: true`
- "Execution" selected → `use_for_execution: true`
- Unselected → `false`

**If "No":** Set `agent_teams.enabled: false` (all sub-toggles remain at defaults).
</step>

<step name="update_config">
Merge new settings into existing config.json:

```json
{
  ...existing_config,
  "model_profile": "quality" | "balanced" | "budget",
  "workflow": {
    "research": true/false,
    "plan_check": true/false,
    "verifier": true/false
  },
  "git": {
    "branching_strategy": "none" | "phase" | "milestone"
  }
}
```

**If Agent Teams step was shown**, also merge:

```json
{
  "agent_teams": {
    "enabled": true/false,
    "use_for_research": true/false,
    "use_for_planning": true/false,
    "use_for_execution": true/false,
    "teammate_mode": "in-process"
  }
}
```

Write updated config to `.planning/config.json`.
</step>

<step name="confirm">
Display:

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 GSD ► SETTINGS UPDATED
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

| Setting              | Value |
|----------------------|-------|
| Model Profile        | {quality/balanced/budget} |
| Plan Researcher      | {On/Off} |
| Plan Checker         | {On/Off} |
| Execution Verifier   | {On/Off} |
| Git Branching        | {None/Per Phase/Per Milestone} |
```

**If Agent Teams step was shown, also include:**

```
| Agent Teams          | {On/Off} |
| Teams: Research      | {On/Off} |
| Teams: Planning      | {On/Off} |
| Teams: Execution     | {On/Off} |
```

```
These settings apply to future /gsd:plan-phase and /gsd:execute-phase runs.

Quick commands:
- /gsd:set-profile <profile> — switch model profile
- /gsd:plan-phase --research — force research
- /gsd:plan-phase --skip-research — skip research
- /gsd:plan-phase --skip-verify — skip plan check
```
</step>

</process>

<success_criteria>
- [ ] Current config read
- [ ] User presented with 5 settings (profile + 3 workflow toggles + git branching)
- [ ] If CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1: Agent Teams settings presented
- [ ] Config updated with model_profile, workflow, git, and optionally agent_teams sections
- [ ] Changes confirmed to user
</success_criteria>
