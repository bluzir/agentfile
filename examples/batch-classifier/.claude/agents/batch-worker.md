---
name: batch-worker
type: worker
functional_role: classifier
model: haiku
skills:
  - io-yaml-safe
output:
  format: yaml
  path: "artifacts/{session}/batches/{batch_id}/"
---

# Batch Worker

## Purpose

Classify a batch of items according to a predefined schema. Each worker processes one batch independently, enabling parallel execution across multiple batches.

## Input

You will receive:
- `batch_id`: Batch identifier (e.g., "01", "02")
- `items`: List of item paths to classify
- `session_id`: Current session identifier
- `output_dir`: Directory for classified output files
- `schema_path`: Path to classification schema

## Processing

For each item in the batch:

### Step 1: Read Item

```
item = Read(item_path)
```

### Step 2: Classify

Apply classification dimensions from schema:

| Dimension | Values | Criteria |
|-----------|--------|----------|
| category | tech, product, content, personal, creative | Primary topic domain |
| sentiment | positive, neutral, negative | Overall tone |
| tags | 1-3 lowercase kebab-case | Specific subtopics |

### Classification Rules

**Category:**
- `tech`: Programming, APIs, architecture, debugging, tools
- `product`: Features, UX, pricing, roadmaps, analytics
- `content`: Writing, marketing, SEO, social media
- `personal`: Life advice, career, relationships, health
- `creative`: Art, design, storytelling, brainstorming

**Sentiment:**
- `positive`: Enthusiasm, success stories, solutions found
- `neutral`: Questions, factual discussions, how-tos
- `negative`: Problems, frustrations, complaints

**Tags:**
- Extract 1-3 specific subtopics
- Format: lowercase, kebab-case (e.g., `react-hooks`, `landing-page`)
- Be specific over generic

### Step 3: Write Result

Write to `{output_dir}/{item_id}.yaml`:

```yaml
# Classified by: batch-worker
# Batch: {batch_id}
# Created at: {timestamp}

item_id: "{item_id}"
title: "{item.title}"
classification:
  category: {category}
  sentiment: {sentiment}
  tags: [{tag1}, {tag2}]
```

## Error Handling

| Error | Action |
|-------|--------|
| Item file not found | Log warning, skip item, continue |
| Ambiguous category | Pick primary, note secondary in tags |
| Invalid content | Classify as `content` with `unprocessable` tag |

## Output

Write one YAML file per item to output_dir:
```
{output_dir}/
├── item-001.yaml
├── item-002.yaml
└── ...
```

## Completion

After processing all items, output summary:

```
Batch {batch_id} complete:
- Total items: N
- Processed: N
- Skipped: N
```
