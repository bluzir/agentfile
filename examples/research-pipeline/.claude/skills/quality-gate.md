---
name: quality-gate
type: atomic
version: v1.0
description: "Evaluate synthesis quality, return PASS/WARN/FAIL verdict"
input:
  required:
    - synthesis_path
output:
  type: verdict
  schema: quality.yaml
---

# Quality Gate

## Purpose

Evaluate research synthesis quality against thresholds. Return verdict that determines whether to proceed to report generation.

## Input

| Parameter | Type | Description |
|-----------|------|-------------|
| `synthesis_path` | string | Path to synthesis.yaml |

## Procedure

### Step 1: Load Synthesis

```
synthesis = Read(synthesis_path)
metrics = synthesis.quality_metrics
```

### Step 2: Evaluate Metrics

| Metric | Threshold | Weight | Evaluation |
|--------|-----------|--------|------------|
| saturation | ≥50 | 0.3 | Information completeness |
| diversity | ≥0.5 | 0.2 | Source variety |
| tier_quality | ≥0.6 | 0.3 | Average source quality |
| evidence_depth | ≥3 | 0.2 | Findings per insight |

For each metric:
```
score = metric_value >= threshold ? 1.0 : metric_value / threshold
weighted_score = score * weight
```

### Step 3: Check Critical Failures

**Automatic FAIL:**
- saturation < 30 (insufficient coverage)
- total_sources < 10 (not enough research)
- insights.length < 3 (not enough synthesis)
- S+A tier sources < 3 (quality too low)

### Step 4: Calculate Verdict

```
total_score = sum(weighted_scores)

PASS: total_score >= 0.8 AND no critical failures
WARN: total_score >= 0.5 AND no critical failures
FAIL: total_score < 0.5 OR any critical failure
```

### Step 5: Generate Issues & Recommendations

For each failed metric:
```yaml
issue: "Saturation below threshold (42% vs 50%)"
recommendation: "Research additional aspects or expand queries"
severity: warning|critical
```

## Output Schema

```yaml
verdict: PASS|WARN|FAIL
total_score: number  # 0-1
evaluated_at: timestamp

scores:
  saturation:
    value: number
    threshold: number
    passed: boolean
    weight: number
    weighted_score: number
  diversity:
    # same structure
  tier_quality:
    # same structure
  evidence_depth:
    # same structure

critical_checks:
  min_saturation: boolean
  min_sources: boolean
  min_insights: boolean
  min_quality_sources: boolean

issues:
  - metric: string
    issue: string
    severity: warning|critical

recommendations:
  - string
```

## Decision Table

| Condition | Verdict | Action |
|-----------|---------|--------|
| total ≥0.8, no criticals | PASS | Proceed to report |
| total ≥0.5, no criticals | WARN | Proceed with caveats |
| total <0.5 | FAIL | Identify gaps, suggest re-research |
| any critical failure | FAIL | Address critical issue first |

## Example Output

### PASS Example
```yaml
verdict: PASS
total_score: 0.87
evaluated_at: "2026-01-30T10:35:00Z"

scores:
  saturation:
    value: 72
    threshold: 50
    passed: true
    weight: 0.3
    weighted_score: 0.3
  diversity:
    value: 0.75
    threshold: 0.5
    passed: true
    weight: 0.2
    weighted_score: 0.2
  tier_quality:
    value: 0.68
    threshold: 0.6
    passed: true
    weight: 0.3
    weighted_score: 0.3
  evidence_depth:
    value: 4.2
    threshold: 3
    passed: true
    weight: 0.2
    weighted_score: 0.2

critical_checks:
  min_saturation: true
  min_sources: true
  min_insights: true
  min_quality_sources: true

issues: []
recommendations: []
```

### WARN Example
```yaml
verdict: WARN
total_score: 0.62

scores:
  saturation:
    value: 48
    threshold: 50
    passed: false
    weight: 0.3
    weighted_score: 0.288
  # ...

issues:
  - metric: saturation
    issue: "Saturation slightly below threshold (48% vs 50%)"
    severity: warning

recommendations:
  - "Consider expanding research on underrepresented aspects"
  - "Report will note limited coverage in some areas"
```

### FAIL Example
```yaml
verdict: FAIL
total_score: 0.35

critical_checks:
  min_saturation: false  # Critical failure

issues:
  - metric: saturation
    issue: "Critical: Saturation far below minimum (25% vs 30%)"
    severity: critical
  - metric: tier_quality
    issue: "Source quality below threshold (0.45 vs 0.6)"
    severity: warning

recommendations:
  - "Research coverage is insufficient for synthesis"
  - "Re-run Phase 2 with expanded queries"
  - "Consider adding more aspects to plan"
```
