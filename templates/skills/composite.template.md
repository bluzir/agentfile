---
name: {skill-name}
type: composite
version: v1.0
description: "{Description combining multiple operations}"
depends:
  - {atomic-skill-1}
  - {atomic-skill-2}
input:
  required:
    - {input}
output:
  type: {verdict|data}
  schema: {output}.schema.yaml
---

# {Skill Name}

## Purpose

{What this composite skill achieves by combining atomic skills.}

## Components

| Skill | Type | Purpose in Composition |
|-------|------|------------------------|
| `{atomic-skill-1}` | atomic | {Role} |
| `{atomic-skill-2}` | atomic | {Role} |
| `{atomic-skill-3}` | atomic | {Role} |

## Input

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `{input}` | {type} | Yes | {Description} |

## Procedure

### Step 1: {First Atomic Skill}

Invoke `{atomic-skill-1}`:
- Input: `{input field}`
- Output: `{intermediate_1}`

```yaml
# Intermediate result
{field}: {value}
```

### Step 2: {Second Atomic Skill}

Invoke `{atomic-skill-2}`:
- Input: `{intermediate_1}` from Step 1
- Output: `{intermediate_2}`

```yaml
# Intermediate result
{field}: {value}
```

### Step 3: Combine Results

Merge outputs from Steps 1-2:
- Apply combination logic
- Generate final verdict/data

## Decision Logic

```
IF {atomic-skill-1}.result == {condition_1}
  AND {atomic-skill-2}.result == {condition_2}
THEN final_verdict = {PASS|FAIL|...}

IF {atomic-skill-1}.result == {condition_3}
  OR {atomic-skill-2}.result == {condition_4}
THEN final_verdict = {WARN}

ELSE final_verdict = {DEFAULT}
```

## Output

### Schema

```yaml
verdict: PASS|WARN|FAIL|EXCLUDE
confidence: float  # 0.0-1.0
components:
  {skill-1}:
    result: {value}
    weight: {float}
  {skill-2}:
    result: {value}
    weight: {float}
combined_score: float
reason: string
```

### Example

```yaml
verdict: PASS
confidence: 0.85
components:
  tier-weights:
    result: A
    weight: 0.8
  slop-check:
    result: 22
    weight: 0.78
combined_score: 0.79
reason: "A-tier source with low AI content score"
```

## Weights & Thresholds

| Component | Weight | Threshold | Impact |
|-----------|--------|-----------|--------|
| `{skill-1}` | {0.X} | {value} | {High/Medium/Low} |
| `{skill-2}` | {0.X} | {value} | {High/Medium/Low} |

### Combination Formula

```
combined_score = (skill_1.weight × skill_1.normalized_score)
               + (skill_2.weight × skill_2.normalized_score)

verdict =
  PASS    if combined_score >= 0.7
  WARN    if combined_score >= 0.4
  FAIL    if combined_score >= 0.2
  EXCLUDE if combined_score < 0.2
```

## Edge Cases

| Case | Component Results | Final Verdict |
|------|-------------------|---------------|
| High tier, high slop | tier=S, slop=85 | WARN (quality tier, but AI-generated) |
| Low tier, low slop | tier=D, slop=10 | FAIL (authentic but low quality) |
| X-tier (any slop) | tier=X, slop=* | EXCLUDE (skip entirely) |

## Usage Example

```
Input:
  url: "https://example.com/article"

Step 1 (tier-weights):
  tier: A
  weight: 0.8

Step 2 (slop-check):
  score: 22
  weight: 0.78

Combined:
  verdict: PASS
  confidence: 0.85
  combined_score: 0.79
  reason: "A-tier source with low AI content score"
```
