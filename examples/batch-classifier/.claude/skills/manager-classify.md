---
name: manager-classify
type: manager
version: v1.0
description: "Batch classification pipeline orchestration"
---

# Batch Classification Manager

## Purpose

Orchestrate multi-phase classification pipeline from data partitioning to taxonomy generation.

## Overview

```
Phase 1        Phase 2              Phase 3         Phase 4
Partition ───▶ Classify ×N    ───▶ Aggregate  ───▶ Export
   │              │                    │              │
   ▼              ▼                    ▼              ▼
manifest.yaml  batches/*/*.yaml   taxonomy.yaml  summary.md
```

## Prerequisites

Before starting:
- Data directory provided by user
- Session ID generated (format: `classify_{YYYYMMDD}_{random}`)
- artifacts/{session_id}/ directory created

---

## Phase 1: Partition

**Gate:** None (entry point)

**Actions:**
1. Scan data directory for items
2. Split into batches of batch_size items
3. Generate manifest

**Orchestration:**
```
Skill(skill: "partition", args: |
  data_dir: {data_dir}
  batch_size: 50
  session_id: {session}
)
```

**Output:** `artifacts/{session}/batches/manifest.yaml`

```yaml
# manifest.yaml schema
total_items: number
batch_size: number
batches:
  - id: string
    items: string[]  # Paths to items
```

**Quality Check:**
- total_items > 0
- Each batch has <= batch_size items

**Next:** Phase 2

---

## Phase 2: Parallel Classification

**Gate:**
```yaml
type: file_exists
condition: "batches/manifest.yaml"
```

**Actions:**
1. Read manifest
2. Spawn batch-worker for each batch
3. Wait for all to complete

**Orchestration:**
```
manifest = Read("artifacts/{session}/batches/manifest.yaml")

# Create output directories
For each batch in manifest.batches:
  Bash("mkdir -p artifacts/{session}/batches/{batch.id}")

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

**Output:** `artifacts/{session}/batches/{batch_id}/*.yaml`

**Quality Check:**
- All batches have output files
- count(output files) ~= manifest.total_items

**Next:** Phase 3

---

## Phase 3: Aggregation

**Gate:**
```yaml
type: files_exist
condition: "batches/*/*.yaml"
```

**Actions:**
1. Load all classified items
2. Build taxonomy with counts
3. Generate statistics

**Orchestration:**
```
Skill(skill: "taxonomy-builder", args: |
  session_id: {session}
  batches_dir: artifacts/{session}/batches/
  output_path: artifacts/{session}/taxonomy.yaml
)
```

**Output:** `artifacts/{session}/taxonomy.yaml`

```yaml
# taxonomy.yaml schema
metadata:
  session_id: string
  total_items: number
  created_at: timestamp

categories:
  tech:
    count: number
    percentage: number
  product:
    count: number
    percentage: number
  # ...

sentiments:
  positive:
    count: number
    percentage: number
  # ...

tags:
  - name: string
    count: number
  # sorted by count descending

cross_tabulation:
  # category × sentiment matrix
  tech:
    positive: number
    neutral: number
    negative: number
  # ...
```

**Next:** Phase 4

---

## Phase 4: Export

**Gate:**
```yaml
type: file_exists
condition: "taxonomy.yaml"
```

**Actions:**
1. Read taxonomy
2. Generate summary report
3. Update state to completed

**Orchestration:**
```
taxonomy = Read("artifacts/{session}/taxonomy.yaml")

# Generate markdown summary
Write("artifacts/{session}/summary.md", |
  # Classification Summary

  **Session:** {session}
  **Total Items:** {taxonomy.metadata.total_items}
  **Generated:** {timestamp}

  ## Categories

  | Category | Count | % |
  |----------|-------|---|
  {for each category}

  ## Sentiment Distribution

  | Sentiment | Count | % |
  |-----------|-------|---|
  {for each sentiment}

  ## Top Tags

  {top 20 tags by count}

  ## Cross-Tabulation

  {category × sentiment matrix}
)
```

**Output:** `artifacts/{session}/summary.md`

**Next:** None (terminal)

---

## State Management

### State File

Location: `artifacts/{session}/state.yaml`

```yaml
session_id: "classify_20260130_abc123"
data_dir: "data/sample"
workflow: "manager-classify"
current_phase: "classify"
phase_states:
  partition: completed
  classify: in_progress
  aggregate: pending
  export: pending
started_at: "2026-01-30T10:00:00Z"
last_updated: "2026-01-30T10:15:00Z"
error: null
```

### Update Pattern

Before phase:
```yaml
current_phase: "{phase}"
phase_states.{phase}: "in_progress"
last_updated: now()
```

After phase:
```yaml
phase_states.{phase}: "completed"
last_updated: now()
```

---

## Recovery

### On Worker Failure

```
1. Log which batch failed
2. Continue with remaining workers
3. At phase end:
   - Retry failed batches (up to 2 times)
   - If still failing, mark as partial and continue
```

### On Resume

```
1. Read state.yaml
2. Find current_phase
3. If partition complete → check which batches finished
4. Re-run only incomplete batches
5. Continue to aggregation
```

---

## Example Run

```
User: /classify data/sample/

Phase 1: Partition
  ✓ Found 100 items
  ✓ Created 2 batches (50 items each)
  → artifacts/classify_20260130_abc/batches/manifest.yaml

Phase 2: Classification
  ✓ Spawning 2 workers in parallel
  ✓ batches/01/ (50 items classified)
  ✓ batches/02/ (50 items classified)

Phase 3: Aggregation
  ✓ Built taxonomy from 100 items
  ✓ 5 categories, 3 sentiments, 47 unique tags
  → artifacts/classify_20260130_abc/taxonomy.yaml

Phase 4: Export
  ✓ Generated summary report
  → artifacts/classify_20260130_abc/summary.md

Status: COMPLETED
```
