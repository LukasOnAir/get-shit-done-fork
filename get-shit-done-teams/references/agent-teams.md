<agent_teams_integration>

Reference for GSD's optional Agent Teams integration. Agent Teams is an experimental Claude Code feature (Feb 2026) that enables independent teammate sessions with inter-agent messaging and shared task lists.

<detection>

**Two conditions must BOTH be true for Agent Teams to activate:**

1. **Environment:** `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1`
2. **Config:** `agent_teams.enabled: true` in `.planning/config.json`

Detection is exposed via init commands as `teams_available` (boolean). Workflows check this field — never check the env var inline.

```javascript
// In gsd-teams-tools.js:
function detectAgentTeams() {
  return process.env.CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS === '1';
}

// Init commands return:
// teams_available: detectAgentTeams() && config.agent_teams.enabled
// teams_config: { enabled, use_for_<workflow>, teammate_mode }
```

</detection>

<integration_patterns>

## Pattern 1: Research Debate (new-project, new-milestone)

**Standard flow:** 4 parallel Task() researchers write files independently, then synthesizer combines.

**Teams flow:** 4 researcher teammates write files, then read each other's work, message challenges for contradictions, and update their own files if convinced. Synthesizer stays as Task() — doesn't need inter-agent comms.

**Key addition to each researcher's prompt:**
```
<team_behavior>
After writing your research file:
1. Read the other 3 researchers' output files
2. Look for contradictions, gaps, or misalignments
3. Message teammates about specific issues
4. Update your own file if a teammate's challenge is convincing
5. Mark task completed only after cross-checking
</team_behavior>
```

**Value:** Higher quality research through adversarial review. Researchers don't just work in parallel — they challenge each other.

## Pattern 2: Plan-Check Revision Loop (plan-phase)

**Standard flow:** Sequential Task(planner) -> Task(checker) -> if issues -> Task(planner revision) -> Task(checker) -> repeat up to 3x.

**Teams flow:** Planner and checker as teammates with direct messaging. Planner creates plans, messages checker "ready." Checker verifies, messages back result. If issues, planner revises and messages "re-check." Loop up to 3 iterations via direct messaging — no lead involvement needed.

**Coordination:** Via messaging, NOT task dependencies. The planner and checker need to interact during execution (revisions), so `blockedBy` would be wrong.

**Value:** Faster iteration — no lead context overhead per round-trip. Planner and checker negotiate directly.

## Pattern 3: Self-Claiming Execution (execute-phase)

**Standard flow:** Wave-by-wave execution. Wait for all plans in wave N to complete before starting wave N+1. Idle time when one plan finishes early.

**Teams flow:** All plan tasks created upfront with `blockedBy` mapping `depends_on` from PLAN.md frontmatter. Generic executor teammates claim available tasks, execute, mark complete, claim next. No wave idle time.

**Key constraints:**
- Lead handles ALL STATE.md updates (prevents concurrent write conflicts)
- Executors must NOT update STATE.md
- Checkpoints relayed through lead: executor messages lead -> lead presents to user -> lead messages back

**Value:** Most impactful integration. Eliminates wave idle time entirely. Plans start executing as soon as their dependencies are satisfied.

</integration_patterns>

<constraints>

| Constraint | Why | Enforcement |
|-----------|-----|-------------|
| No nested teams | Agent Teams doesn't support teams within teams | Synthesizer stays as Task(), not teammate |
| Windows compatibility | Many users on Windows, no tmux | `teammate_mode` defaults to `"in-process"` |
| File ownership for STATE.md | Concurrent writes cause git conflicts | Only lead writes STATE.md, executors write SUMMARY.md |
| Checkpoint relay | Executors can't interact with user directly | Executor messages lead, lead presents to user, messages back |
| Fallback required | Teams is experimental, may not be available | Every workflow MUST have standard Task() path as default |

</constraints>

<fallback_pattern>

Every workflow that supports Agent Teams MUST follow this pattern:

```
**If `teams_available` AND `teams_config.use_for_<workflow>`:**

  <agent_teams_<workflow>_path>
    [Teams implementation]
  </agent_teams_<workflow>_path>

**Otherwise (default):**

  <standard_<workflow>_path>
    [Existing Task() implementation, unchanged]
  </standard_<workflow>_path>
```

The standard path must be IDENTICAL to pre-Teams behavior. No regressions when Teams is disabled.

</fallback_pattern>

<config_reference>

```json
"agent_teams": {
  "enabled": false,          // Master toggle (off by default)
  "use_for_research": true,  // Research workflows (new-project, new-milestone)
  "use_for_planning": true,  // Plan-check revision loop (plan-phase)
  "use_for_execution": true, // Self-claiming execution (execute-phase)
  "teammate_mode": "in-process"  // Windows-compatible default
}
```

Configure via `/gsd-teams:settings` (shows Agent Teams options only when env var is set) or directly edit `.planning/config.json`.

</config_reference>

</agent_teams_integration>
