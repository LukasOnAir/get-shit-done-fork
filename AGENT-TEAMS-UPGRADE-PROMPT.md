# GSD Agent Teams Upgrade — Starting Prompt

Copy everything below the line into a new Claude Code session opened in this repo.

---

## Context

I'm working on a fork of **GSD (Get Shit Done)** — a meta-prompting, context engineering, and spec-driven development system for Claude Code. It's at `D:\2. Git\get-shit-done-fork`.

Claude Code just shipped **Agent Teams** with Opus 4.6 (Feb 5, 2026). GSD doesn't leverage this yet — it's built entirely on the `Task()` subagent pattern. I want to upgrade GSD to optionally use Agent Teams where it adds value.

## What GSD Is

GSD solves "context rot" (quality degradation as context fills up) via a **thin orchestrator + specialized subagents** pattern. Key architecture:

- **Commands** (`commands/gsd-teams/*.md`) — User-facing slash commands that invoke workflows
- **Workflows** (`get-shit-done-teams/workflows/*.md`) — Orchestrator scripts that coordinate agent spawning
- **Agents** (`agents/*.md`) — 11 specialized subagent definitions:
  - `gsd-teams-planner` — Creates PLAN.md files
  - `gsd-teams-executor` — Executes plans with atomic commits
  - `gsd-teams-verifier` — Confirms deliverables match requirements
  - `gsd-teams-phase-researcher` — Researches implementation approaches
  - `gsd-teams-project-researcher` — Investigates domain ecosystems (4 spawned in parallel)
  - `gsd-teams-research-synthesizer` — Aggregates research findings
  - `gsd-teams-plan-checker` — Verifies plans achieve phase goals
  - `gsd-teams-debugger` — Diagnoses execution failures
  - `gsd-teams-codebase-mapper` — Analyzes existing codebases
  - `gsd-teams-roadmapper` — Converts requirements to phases
  - `gsd-teams-integration-checker` — Validates integrations
- **Templates** (`get-shit-done-teams/templates/`) — Document templates for output
- **References** (`get-shit-done-teams/references/`) — Shared knowledge files
- **gsd-teams-tools.js** (`get-shit-done-teams/bin/gsd-teams-tools.js`) — Central CLI for state management, git ops, config

**State is file-based:**
- `.planning/STATE.md` — Short-term memory across sessions
- `.planning/config.json` — Workflow configuration
- `.planning/ROADMAP.md` — Phase breakdown
- `.planning/REQUIREMENTS.md` — Scoped requirements

**Current parallelization:** Wave-based. Plans declare `depends_on` in frontmatter. Orchestrator groups them into waves (topological sort), executes plans within a wave in parallel via multiple `Task()` calls, then moves to the next wave.

**Config schema** (`.planning/config.json`):
```json
{
  "parallelization": {
    "enabled": true,
    "plan_level": true,
    "task_level": false,
    "skip_checkpoints": true,
    "max_concurrent_agents": 3,
    "min_plans_for_parallel": 2
  }
}
```

**Current spawning pattern** (from `execute-phase.md`):
```
Task(
  subagent_type="gsd-teams-executor",
  model="{executor_model}",
  prompt="<objective>Execute plan {plan_number}...</objective>
    <execution_context>@~/.claude/get-shit-done-teams/workflows/execute-plan.md</execution_context>
    <files_to_read>...</files_to_read>"
)
```

Subagents report results back to caller only. No inter-agent communication.

## What Agent Teams Is

Agent Teams (experimental, `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1`) lets one Claude Code session act as a **team lead** that spawns independent **teammates** — each a full Claude Code session with its own context window.

**Key differences from Task() subagents:**

| Aspect | Task() Subagents | Agent Teams |
|--------|-----------------|-------------|
| Communication | Results return to caller only | Teammates message each other directly |
| Coordination | Parent manages all work | Shared task list + self-coordination |
| Context | Shared parent context | Fully independent sessions |
| Self-claim | No | Teammates pick up next unblocked task |
| Inter-agent debate | No | Teammates can challenge each other |
| Cost | Lower | Higher (each teammate = separate instance) |

**Architecture components:**
- **Team lead** — Creates team, spawns teammates, coordinates
- **Teammates** — Independent Claude Code sessions
- **Shared task list** — `TaskCreate`/`TaskUpdate`/`TaskList` (GSD already uses these!)
- **Mailbox** — Direct messaging between agents

**TeammateTool API (13 operations):**
- Team: `spawnTeam`, `discoverTeams`, `requestJoin`, `approveJoin`, `rejectJoin`
- Comms: `write` (direct message), `broadcast` (all teammates)
- Lifecycle: `requestShutdown`, `approveShutdown`, `rejectShutdown`, `cleanup`
- Planning: `approvePlan`, `rejectPlan`

**Environment variables set for teammates:**
- `CLAUDE_CODE_TEAM_NAME`, `CLAUDE_CODE_AGENT_ID`, `CLAUDE_CODE_AGENT_NAME`
- `CLAUDE_CODE_AGENT_TYPE`, `CLAUDE_CODE_AGENT_COLOR`
- `CLAUDE_CODE_PLAN_MODE_REQUIRED`, `CLAUDE_CODE_PARENT_SESSION_ID`

**Quality hooks:**
- `TeammateIdle` — Runs when teammate about to go idle (exit code 2 = send feedback, keep working)
- `TaskCompleted` — Runs when task being marked complete (exit code 2 = prevent completion)

**Teammates auto-load:** CLAUDE.md, MCP servers, skills from `.claude/skills/`. They do NOT inherit the lead's conversation history.

**Key constraint:** No nested teams — teammates cannot spawn their own teams.

**Storage:**
- Team config: `~/.claude/teams/{team-name}/config.json`
- Task list: `~/.claude/tasks/{team-name}/`
- Messages: `~/.claude/teams/{team}/inboxes/{agent}.json`

**Docs:** https://code.claude.com/docs/en/agent-teams

## The Upgrade Goal

Add optional Agent Teams support to GSD. This should be:

1. **Opt-in** — New config option, off by default. GSD must still work perfectly with Task() subagents.
2. **Pragmatic** — Use Teams where it adds genuine value (inter-agent debate, self-coordination), keep Task() where it's sufficient (simple focused work).
3. **True to GSD's philosophy** — No enterprise theater. No complexity for complexity's sake. The user experience should stay simple.

## Where Teams Adds the Most Value

### High-value targets (implement these):

1. **Research phase** (`new-project.md` workflow) — Currently spawns 4 `gsd-teams-project-researcher` subagents in parallel (Stack, Features, Architecture, Pitfalls), then a separate `gsd-teams-research-synthesizer`. With Teams, researchers could **challenge each other's findings** (the "competing hypotheses" pattern Teams was designed for), and synthesis could happen through debate rather than a separate agent.

2. **Plan-checker revision loop** (`plan-phase.md` workflow) — Currently: `gsd-teams-planner` → `gsd-teams-plan-checker` → back to planner (up to 3 iterations), all orchestrator-managed. With Teams, planner and checker could be teammates that message each other directly, reducing orchestrator context load.

3. **Execution phase** (`execute-phase.md` workflow) — Currently: wave-based with orchestrator grouping and spawning. With Teams, executor teammates could self-claim tasks from the shared task list. The dependency system in Teams automatically blocks tasks until dependencies complete. The lead (orchestrator) stays lean.

### Keep as Task() subagents (don't convert):

- `gsd-teams-codebase-mapper` — Focused, report-back work. No inter-agent value.
- `gsd-teams-debugger` — Interactive, needs user involvement. Not a team pattern.
- `gsd-teams-roadmapper` — Single agent, no parallelism needed.
- `gsd-teams-research-synthesizer` — May be replaced by researcher debate pattern, but keep as fallback.

## Implementation Approach

### 1. Config extension

Add to `.planning/config.json` schema and `get-shit-done-teams/templates/config.json`:

```json
{
  "agent_teams": {
    "enabled": false,
    "use_for_research": true,
    "use_for_planning": true,
    "use_for_execution": true,
    "teammate_mode": "auto",
    "default_team_size": 4
  }
}
```

### 2. Detection and fallback

Workflows should detect whether Agent Teams is available (check `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS` env var) and fall back to Task() subagents if not. Never break the existing flow.

### 3. Workflow changes needed

**`get-shit-done-teams/workflows/new-project.md`:**
- When teams enabled: spawn a team with 4 researcher teammates + instruct them to challenge each other
- Researchers share findings via messaging, converge on synthesis
- May eliminate the need for a separate synthesizer agent
- Fallback: current 4x `Task()` + synthesizer pattern

**`get-shit-done-teams/workflows/plan-phase.md`:**
- When teams enabled: spawn planner + checker as teammates
- They message each other directly for the revision loop
- Orchestrator just monitors completion instead of managing iterations
- Fallback: current sequential spawn pattern

**`get-shit-done-teams/workflows/execute-phase.md`:**
- When teams enabled: spawn executor teammates, create tasks from plan inventory
- Tasks include dependency info so blocked tasks auto-wait
- Teammates self-claim when done with current task
- Lead spot-checks via SUMMARYs as today
- Fallback: current wave-based Task() pattern

### 4. gsd-teams-tools.js changes

- Add `agent_teams` config parsing to `init` commands
- Return `teams_available` boolean in init JSON (check env var)
- Return `teams_config` object in init JSON

### 5. Quality gates

Consider adding hooks:
- `TeammateIdle` hook that checks if there are remaining unclaimed tasks
- `TaskCompleted` hook that runs spot-checks (file existence, git commits) before allowing completion

### 6. Documentation

- Update `get-shit-done-teams/workflows/settings.md` with the new `agent_teams` config options
- Update `get-shit-done-teams/references/` with a new `agent-teams.md` reference doc

## Important Constraints

- **Windows compatibility** — Split-pane mode (tmux) doesn't work on Windows Terminal. In-process mode works. Don't assume tmux availability.
- **No nested teams** — Teammates can't spawn sub-teams. The lead must own all spawning.
- **File conflicts** — Two executor teammates editing the same file will cause overwrites. The existing plan system already assigns files to specific plans, so this should be manageable, but verify.
- **Cost awareness** — Teams uses significantly more tokens. Make it opt-in and document the tradeoff.
- **Experimental API** — Agent Teams is still experimental. Keep the implementation behind a feature flag and design for graceful degradation.

## Your Task

1. **Read the codebase** — Start by reading the key files: workflows (`execute-phase.md`, `new-project.md`, `plan-phase.md`), agents (especially `gsd-teams-executor.md`, `gsd-teams-project-researcher.md`, `gsd-teams-planner.md`, `gsd-teams-plan-checker.md`), `gsd-teams-tools.js`, and the config template.
2. **Read the Agent Teams docs** — Fetch https://code.claude.com/docs/en/agent-teams for the latest API details.
3. **Plan the implementation** — Use `/gsd-teams:new-project` or enter plan mode to design the changes.
4. **Implement incrementally** — Start with config + detection, then research workflow, then planning workflow, then execution workflow. Test each step.
5. **Preserve backward compatibility** — Every workflow must still work identically when `agent_teams.enabled` is false (the default).
