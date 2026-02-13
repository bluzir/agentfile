# OpenClaw Compatibility

How to run claude-pipe pipelines on [OpenClaw](https://github.com/nicholasgriffintn/openclaw) without losing parallelism.

> **TL;DR:** OpenClaw executes tool calls sequentially (`maxConcurrent: 1`). claude-pipe fan-out phases use parallel `Task(run_in_background: true)` calls. Replace those with OpenClaw's native `sessions_spawn` to restore parallel execution.

---

## The Problem

Claude Code sends multiple `Task` calls in a single message — they run in parallel. OpenClaw processes tool calls one at a time through `CommandLane.Main` in `command-queue.ts`.

```
Claude Code (parallel):

  ┌──────────────────────────────────────────────┐
  │  Task(researcher-1)  ████████████████  done  │
  │  Task(researcher-2)  ████████████████  done  │   ~15 min
  │  Task(researcher-3)  ████████████████  done  │
  │  Task(researcher-4)  ████████████████  done  │
  │  Task(researcher-5)  ████████████████  done  │
  └──────────────────────────────────────────────┘

OpenClaw (sequential, maxConcurrent: 1):

  ┌──────────────────────────────────────────────────────────────────────────────────┐
  │  Task(r-1) ████  Task(r-2) ████  Task(r-3) ████  Task(r-4) ████  Task(r-5) ████│  ~75 min
  └──────────────────────────────────────────────────────────────────────────────────┘
```

### Impact

| Pipeline | Workers | Parallel Time | Sequential Time | Slowdown |
|----------|---------|---------------|-----------------|----------|
| Research (medium) | 5 aspects | ~15 min | ~75 min | 5× |
| Research (deep) | 7 aspects | ~15 min | ~105 min | 7× |
| Batch classifier | N batches | ~10 min | ~10×N min | N× |

The bottleneck is `maxConcurrent: 1` in OpenClaw's command queue. Only fan-out phases are affected — sequential phases (planning, synthesis, quality gate, export) work identically.

---

## Strategies

| Strategy | Effort | Risk | Recommended |
|----------|--------|------|-------------|
| **A. `sessions_spawn`** | Low | Low | **Yes** |
| B. Increase `maxConcurrent` | Low | High | No |
| C. `Promise.all()` upstream | High | Medium | Long-term |

---

## Strategy A: `sessions_spawn`

OpenClaw has native support for spawning parallel sessions via `sessions_spawn`. This maps directly to claude-pipe's fan-out pattern.

### Research Pipeline — Phase 2

**Before** (claude-pipe, from `manager-research.md` lines 96-121):

```
plan = Read("artifacts/{session}/plan.yaml")

# Spawn all researchers in parallel (single message, multiple Task calls)
For each aspect in plan.aspects:
  Task(
    subagent_type: "general-purpose",
    prompt: |
      Load aspect-researcher agent from .claude/agents/aspect-researcher.md

      Research this aspect:
      - aspect_id: {aspect.id}
      - aspect_name: {aspect.name}
      - aspect_description: {aspect.description}
      - queries: {aspect.queries}
      - session_id: {session}
      - output_path: artifacts/{session}/aspects/{aspect.id}.yaml

      Write findings to the output path.
    description: "Research {aspect.name}",
    run_in_background: true
  )

# Wait for all
For each task_id in spawned_tasks:
  TaskOutput(task_id: task_id, block: true)
```

**After** (OpenClaw):

```
plan = Read("artifacts/{session}/plan.yaml")

# Spawn all researchers in parallel via sessions_spawn
For each aspect in plan.aspects:
  sessions_spawn(
    task: |
      Load aspect-researcher agent from .claude/agents/aspect-researcher.md

      Research this aspect:
      - aspect_id: {aspect.id}
      - aspect_name: {aspect.name}
      - aspect_description: {aspect.description}
      - queries: {aspect.queries}
      - session_id: {session}
      - output_path: artifacts/{session}/aspects/{aspect.id}.yaml

      Write findings to the output path.
    agentId: "researcher-{aspect.id}"
  )

# Poll until all complete
Loop:
  For each agentId in spawned_agents:
    result = process(action: "poll", agentId: agentId)
    if result.status == "completed":
      mark as done
  if all done: break
  wait(30s)
```

### Batch Classifier — Phase 2

**Before** (claude-pipe, from `manager-classify.md` lines 84-113):

```
manifest = Read("artifacts/{session}/batches/manifest.yaml")

# Spawn ALL workers in parallel (single message, multiple Task calls)
For each batch in manifest.batches:
  Task(
    subagent_type: "general-purpose",
    prompt: |
      Load batch-worker agent from .claude/agents/batch-worker.md

      Classify this batch:
      - batch_id: {batch.id}
      - items: {batch.items}
      - session_id: {session}
      - output_dir: artifacts/{session}/batches/{batch.id}/
      - schema_path: .claude/schemas/classified-item.yaml

      Read each item file, classify it, and write result to output_dir.
    description: "Classify batch {batch.id}",
    run_in_background: true
  )

# Wait for all workers to complete
For each task_id in spawned_tasks:
  TaskOutput(task_id: task_id, block: true)
```

**After** (OpenClaw):

```
manifest = Read("artifacts/{session}/batches/manifest.yaml")

# Spawn ALL workers in parallel via sessions_spawn
For each batch in manifest.batches:
  sessions_spawn(
    task: |
      Load batch-worker agent from .claude/agents/batch-worker.md

      Classify this batch:
      - batch_id: {batch.id}
      - items: {batch.items}
      - session_id: {session}
      - output_dir: artifacts/{session}/batches/{batch.id}/
      - schema_path: .claude/schemas/classified-item.yaml

      Read each item file, classify it, and write result to output_dir.
    agentId: "classifier-{batch.id}"
  )

# Poll until all complete
Loop:
  For each agentId in spawned_agents:
    result = process(action: "poll", agentId: agentId)
    if result.status == "completed":
      mark as done
  if all done: break
  wait(30s)
```

### Primitive Mapping

| claude-pipe | OpenClaw | Notes |
|-------------|----------|-------|
| `Task(run_in_background: true)` | `sessions_spawn(task, agentId)` | Parallel worker spawn |
| `TaskOutput(task_id, block: true)` | `process(action: "poll", agentId)` | Poll loop replaces blocking wait |
| `Task(run_in_background: false)` | `sessions_spawn` + immediate poll | Sequential tasks |
| `Skill("skill-name")` | No change | Skills are loaded the same way |
| `Read()` / `Write()` | No change | File I/O is identical |

---

## Strategy B: Increase `maxConcurrent`

Change `maxConcurrent` from `1` to `N` in OpenClaw's `command-queue.ts` (`CommandLane.Main`).

**Pros:** Minimal code change, applies globally.

**Cons:**
- Race conditions — multiple tool calls writing to the same file
- Non-deterministic execution order
- Harder to debug failures
- State corruption if two agents update `state.yaml` simultaneously

**Verdict:** Possible but not recommended. The sequential queue exists for good reasons. Use `sessions_spawn` instead — it provides controlled parallelism with proper session isolation.

---

## Strategy C: `Promise.all()` Upstream

The ideal long-term solution: OpenClaw detects multiple `Task(run_in_background: true)` calls in a single message and executes them concurrently via `Promise.all()`, matching Claude Code's behavior.

This would make claude-pipe pipelines work without modification.

**Status:** Feature request / contribution opportunity. This requires changes to OpenClaw's tool execution layer — specifically, `command-queue.ts` would need to batch background tasks from the same message into a concurrent group while keeping other tool calls sequential.

---

## Adaptation Checklist

1. **Find fan-out phases** — search manager skills for `run_in_background: true`
2. **Replace `Task()` calls** with `sessions_spawn(task, agentId)`
3. **Replace `TaskOutput(block)` calls** with a poll loop using `process(action: "poll")`
4. **Everything else stays the same** — skills, agents, state files, gates, sequential phases

---

## What Stays the Same

| Component | Change Needed? |
|-----------|----------------|
| Agent definitions (`.claude/agents/`) | No |
| Skill definitions (`.claude/skills/`) | No |
| State files (`state.yaml`) | No |
| Data layers (L0 → L1 → L2 → L3) | No |
| Sequential phases (planning, synthesis, quality gate, export) | No |
| Phase gates | No |
| Schemas | No |
| **Manager fan-out blocks** | **Yes** |

Only the fan-out orchestration blocks in manager skills need to change. The workers, skills, state management, and everything downstream remain identical.

---

## Quick Reference

| claude-pipe Primitive | OpenClaw Equivalent |
|-----------------------|---------------------|
| `Task(run_in_background: true)` | `sessions_spawn(task, agentId)` |
| `TaskOutput(task_id, block: true)` | `process(action: "poll", agentId)` loop |
| `Skill("name")` | `Skill("name")` — no change |
| `Read()` / `Write()` | `Read()` / `Write()` — no change |
| `Bash()` | `Bash()` — no change |
| `state.yaml` | `state.yaml` — no change |
| Gates between phases | Gates between phases — no change |
