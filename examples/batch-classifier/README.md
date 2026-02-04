# Batch Classifier Example

A complete batch classification pipeline demonstrating the **"subagent = batch"** pattern for parallel processing of large datasets.

> **Framework Documentation:** See the main [README.md](../../README.md) for framework concepts and conventions.

## Pattern: Subagent = Batch

This example demonstrates how to parallelize work across multiple subagents where each subagent processes one batch of data independently.

```
                            ┌─────────────────┐
                            │   Manager       │
                            │ (orchestrator)  │
                            └────────┬────────┘
                                     │
              ┌──────────────────────┼──────────────────────┐
              │                      │                      │
              ▼                      ▼                      ▼
    ┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
    │  Batch Worker   │    │  Batch Worker   │    │  Batch Worker   │
    │   (batch 01)    │    │   (batch 02)    │    │   (batch N)     │
    └────────┬────────┘    └────────┬────────┘    └────────┬────────┘
             │                      │                      │
             ▼                      ▼                      ▼
    ┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
    │  50 items       │    │  50 items       │    │  remaining      │
    │  classified     │    │  classified     │    │  items          │
    └─────────────────┘    └─────────────────┘    └─────────────────┘
```

### Key Characteristics

- **Partition first**: Split data into fixed-size batches before processing
- **True parallelism**: All batch workers run concurrently (not sequentially)
- **Independent workers**: Each worker processes its batch without dependencies
- **Aggregate last**: Collect results after all workers complete

## Structure

```
batch-classifier/
├── .claude/
│   ├── agents/
│   │   └── batch-worker.md        # Worker: classifies one batch
│   ├── skills/
│   │   ├── manager-classify.md    # Manager: pipeline orchestration
│   │   ├── partition.md           # Atomic: data partitioning
│   │   └── taxonomy-builder.md    # Composite: results aggregation
│   └── schemas/
│       └── classified-item.yaml   # Output schema
├── data/
│   └── sample/                    # Sample items (100 items)
├── artifacts/                     # Runtime outputs (gitignored)
│   └── {session_id}/
│       ├── state.yaml             # Pipeline state
│       ├── batches/
│       │   ├── manifest.yaml      # Partition manifest
│       │   ├── 01/*.yaml          # Batch 1 classified items
│       │   └── 02/*.yaml          # Batch 2 classified items
│       ├── taxonomy.yaml          # Aggregated taxonomy
│       └── summary.md             # Human-readable summary
└── README.md
```

## Running

```
# Classify sample data
/manager-classify data/sample/

# Or with custom batch size
/manager-classify data/sample/ --batch-size 25
```

## Pipeline

```
Phase 1        Phase 2              Phase 3         Phase 4
Partition ───▶ Classify ×N    ───▶ Aggregate  ───▶ Export
   │              │                    │              │
   ▼              ▼                    ▼              ▼
manifest.yaml  batches/*/*.yaml   taxonomy.yaml  summary.md
```

## Phases

| Phase | Agent/Skill | Input | Output |
|-------|-------------|-------|--------|
| 1. Partition | partition | data/*.yaml | batches/manifest.yaml |
| 2. Classify | batch-worker ×N | manifest.batches | batches/{id}/*.yaml |
| 3. Aggregate | taxonomy-builder | batches/*/*.yaml | taxonomy.yaml |
| 4. Export | manager-classify | taxonomy.yaml | summary.md |

## Classification Schema

Each item is classified with:

| Field | Type | Values |
|-------|------|--------|
| category | enum | tech, product, content, personal, creative |
| sentiment | enum | positive, neutral, negative |
| tags | array[1-3] | Specific subtopics in kebab-case |

### Category Definitions

| Category | Description | Examples |
|----------|-------------|----------|
| tech | Programming, APIs, tools | React hooks, API design |
| product | Features, UX, pricing | User onboarding, pricing tiers |
| content | Writing, marketing, SEO | Blog posts, landing pages |
| personal | Career, relationships | Career advice, work-life balance |
| creative | Art, design, storytelling | Logo design, story plots |

## Key Concepts Demonstrated

- **Manager skill** orchestrating from ROOT with phase gates
- **Parallel fan-out** spawning all batch workers in single message
- **Schema validation** ensuring output consistency
- **Aggregation pattern** combining distributed results
- **State persistence** for resume capability

## When to Use This Pattern

**Good fit:**
- Processing 100+ items that can be handled independently
- Each item requires LLM inference (classification, analysis, extraction)
- Results need to be aggregated into summary/taxonomy
- Latency matters (parallel beats sequential)

**Not a fit:**
- Items have dependencies on each other
- Processing requires shared state across items
- Dataset is small (<50 items) - overhead not worth it
- Need streaming results (all-or-nothing completion)

## Customization

To adapt this pipeline:

1. **Change classification schema** - Edit `.claude/schemas/classified-item.yaml`
2. **Modify classification rules** - Update `.claude/agents/batch-worker.md`
3. **Adjust batch size** - Change `batch_size` in partition skill call
4. **Customize aggregation** - Modify `.claude/skills/taxonomy-builder.md`

## Comparison: Batch vs Research Pipeline

| Aspect | Batch Classifier | Research Pipeline |
|--------|------------------|-------------------|
| **Partitioning** | Data (by count) | Semantic (by meaning) |
| **Worker input** | Fixed-size batch of items | Research aspect/topic |
| **Parallelism** | All batches simultaneously | All aspects simultaneously |
| **Aggregation** | Counts & statistics | Synthesis & insights |
| **Use case** | Classification, extraction | Research, analysis |

## Scaling

### More Parallelism

Reduce batch size to create more workers:
```
100 items ÷ 50 batch_size = 2 parallel workers (default)
100 items ÷ 25 batch_size = 4 parallel workers
100 items ÷ 10 batch_size = 10 parallel workers
```

Trade-off: More workers = more overhead, but faster wall-clock time.

### Model Selection

| Model | Use Case | Cost |
|-------|----------|------|
| haiku | High-volume classification (default) | Lowest |
| sonnet | Complex categorization logic | Medium |
| opus | Multi-dimensional analysis | Highest |

### Larger Datasets

For 1000+ items:
1. Consider chunking into multiple runs
2. Add intermediate checkpointing between batches
3. Use haiku model for cost efficiency

## Sample Data Distribution

100 synthetic conversation items:

| Category | Count | Percentage |
|----------|-------|------------|
| tech | 40 | 40% |
| product | 20 | 20% |
| content | 20 | 20% |
| personal | 10 | 10% |
| creative | 10 | 10% |
