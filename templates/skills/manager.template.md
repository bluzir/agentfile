---
name: manager-{workflow}
type: manager
version: v1.0
description: "{Workflow name} orchestration (phases {N}-{M})"
---

# {Workflow Name} Manager

## Purpose

Orchestrate {workflow description} from {start} to {end}. This skill provides workflow instructions for ROOT to execute.

## Overview

```
Phase 1: {Name}     ──→ Phase 2: {Name}     ──→ Phase 3: {Name}
         │                      │                      │
         ▼                      ▼                      ▼
    {output_1}             {output_2}             {output_3}
```

## Prerequisites

Before starting:
- [ ] {Prerequisite 1}
- [ ] {Prerequisite 2}
- [ ] Session ID generated

## Phases

### Phase 1: {Phase Name}

**Gate:** {Entry condition or "None (entry point)"}

**Actions:**
1. {Action 1}
2. {Action 2}
3. {Action 3}

**Orchestration:**
```
# If single agent
Task(
  subagent_type: "general-purpose",
  prompt: "Load {agent-name} agent. {task description}",
  description: "{short description}"
)

# If parallel agents
For each {item} in {collection}:
  Task(
    subagent_type: "general-purpose",
    prompt: "Load {agent-name} agent. Process {item}",
    run_in_background: true
  )
Wait for all tasks
```

**Output:** `artifacts/{session}/phase1-{type}.yaml`

**Quality Check:**
- {Validation criterion 1}
- {Validation criterion 2}

**Next:** Phase 2

---

### Phase 2: {Phase Name}

**Gate:**
```yaml
type: file_exists
condition: "artifacts/{session}/phase1-*.yaml"
min_count: {N}
```

**Actions:**
1. {Action 1}
2. {Action 2}

**Orchestration:**
```
# Parallel worker spawning pattern
aspects = Read("artifacts/{session}/plan.yaml").aspects

For each aspect in aspects:
  Task(
    subagent_type: "general-purpose",
    prompt: |
      Load aspect-researcher agent.
      Research: {aspect.name}
      Queries: {aspect.queries}
      Output: artifacts/{session}/aspects/{aspect.id}.yaml
    run_in_background: true
  )

# Wait for completion
For each task_id:
  TaskOutput(task_id: task_id, block: true)
```

**Output:** `artifacts/{session}/phase2-{type}.yaml`

**Quality Check:**
- {Validation criterion}

**Next:**
- On success → Phase 3
- On failure → {Recovery action}

---

### Phase 3: {Phase Name}

**Gate:**
```yaml
type: quality_threshold
conditions:
  - "count(aspects/*.yaml) >= plan.min_aspects"
  - "all files valid against schema"
```

**Actions:**
1. Load all Phase 2 outputs
2. Run synthesis skill
3. Generate aggregated output

**Orchestration:**
```
Skill(skill: "synthesis")
```

**Output:** `artifacts/{session}/synthesis.yaml`

**Next:** Quality Gate

---

### Quality Gate

**Gate:** `synthesis.yaml` exists

**Actions:**
1. Run quality-gate skill
2. Evaluate verdict

**Orchestration:**
```
Skill(skill: "quality-gate")
verdict = Read("artifacts/{session}/quality.yaml").verdict
```

**Routing:**
| Verdict | Action |
|---------|--------|
| PASS | → Final Phase |
| WARN | → Final Phase (with caveats) |
| FAIL | → Back to Phase 2 with gap analysis |

---

### Final Phase: {Artifact Generation}

**Gate:** Quality gate PASS or WARN

**Actions:**
1. Generate final artifact
2. Update state to completed

**Orchestration:**
```
Task(
  subagent_type: "general-purpose",
  prompt: |
    Load {artifact}-generator agent.
    Generate from: artifacts/{session}/synthesis.yaml
    Output: artifacts/{session}/FINAL_{TYPE}.md
)
```

**Output:** `artifacts/{session}/FINAL_{TYPE}.md`

**Next:** None (terminal)

---

## State Management

### State File

Location: `artifacts/{session}/state.yaml`

```yaml
session_id: "{session_id}"
workflow_id: "manager-{workflow}"
current_phase: "{phase_id}"
phase_states:
  phase_1: pending|in_progress|completed|failed
  phase_2: pending|in_progress|completed|failed
  phase_3: pending|in_progress|completed|failed
  quality_gate: pending|in_progress|completed|failed
  final: pending|in_progress|completed|failed
started_at: "{timestamp}"
last_updated: "{timestamp}"
error: null
```

### State Transitions

```
pending → in_progress  (when phase starts)
in_progress → completed (when phase succeeds)
in_progress → failed    (when phase fails)
failed → in_progress    (on retry)
```

### Update Pattern

Before each phase:
```
state.current_phase = "{phase_id}"
state.phase_states.{phase_id} = "in_progress"
state.last_updated = now()
Write(state)
```

After phase completion:
```
state.phase_states.{phase_id} = "completed"
state.last_updated = now()
Write(state)
```

---

## Recovery

### On Worker Failure

```
If task fails:
  1. Log error to state.yaml
  2. Continue with remaining workers
  3. At phase end, check if minimum outputs exist
  4. If below minimum → phase failed
  5. If above minimum → continue with warning
```

### On Phase Failure

```
1. Update state.phase_states.{phase} = "failed"
2. Update state.error = "{error_message}"
3. Write state
4. Report to user with:
   - What failed
   - What was completed
   - Suggested action
```

### On Resume

```
1. Read state.yaml
2. Find current_phase
3. Check phase_states for last completed
4. Resume from current_phase
5. Skip completed phases
```

---

## Example Session

```
Session: research_20260130_abc123

Phase 1: Planning
  → artifacts/research_20260130_abc123/plan.yaml ✓

Phase 2: Research (parallel)
  → Spawning 5 aspect-researcher workers
  → aspects/tech.yaml ✓
  → aspects/market.yaml ✓
  → aspects/competition.yaml ✓
  → aspects/trends.yaml ✓
  → aspects/risks.yaml ✓

Phase 3: Synthesis
  → artifacts/research_20260130_abc123/synthesis.yaml ✓

Quality Gate
  → PASS (saturation: 78%, diversity: 0.82)

Final: Report Generation
  → artifacts/research_20260130_abc123/FINAL_REPORT.md ✓

Status: COMPLETED
```
