---
name: synthesis
type: composite
version: v1.0
description: "Aggregate findings across aspects, identify patterns and insights"
depends:
  - grounding-protocol
  - anti-cringe
input:
  required:
    - aspects_path
  optional:
    - plan_path
output:
  type: data
  schema: synthesis.yaml
---

# Synthesis

## Purpose

Aggregate findings from multiple aspect research files. Identify cross-aspect patterns, generate insights, calculate quality metrics.

## Components

| Skill | Role |
|-------|------|
| grounding-protocol | Ensure no hallucination in synthesis |
| anti-cringe | Quality filter for insight descriptions |

## Input

| Parameter | Type | Description |
|-----------|------|-------------|
| `aspects_path` | string | Path to aspects/ directory |
| `plan_path` | string | Path to plan.yaml (optional) |

## Procedure

### Step 1: Load All Aspect Files

```
aspects = Glob("artifacts/{session}/aspects/*.yaml")
for each file:
  aspect_data = Read(file)
  collect all findings
```

### Step 2: Deduplicate Findings

Across aspects, findings may overlap. Deduplicate by:
1. Exact URL match → keep higher tier version
2. Semantic similarity → merge, cite both sources

### Step 3: Identify Cross-Aspect Patterns

Look for:
- **Recurring themes:** Same concept in 2+ aspects
- **Contradictions:** Conflicting claims → note both
- **Causal chains:** A (aspect 1) enables B (aspect 2)
- **Gaps:** Expected topics not covered

For each pattern:
```yaml
pattern: "Description of pattern"
type: recurring|contradiction|causal|gap
aspects: ["aspect_1", "aspect_3"]
evidence:
  - finding_id: "f001"
    aspect: "aspect_1"
  - finding_id: "f012"
    aspect: "aspect_3"
strength: 3  # Number of supporting findings
```

### Step 4: Generate Insights

From patterns and strong findings, generate insights:

**Insight criteria:**
- Supported by 2+ sources
- Non-obvious (not just restating a finding)
- Actionable or informative

```yaml
insight:
  id: "i001"
  title: "Short title"
  description: "Clear statement of the insight"
  evidence:
    - finding_id: "f001"
      weight: 0.8
    - finding_id: "f012"
      weight: 0.6
  confidence: high|medium|low
  type: observation|recommendation|warning
```

**Confidence levels:**
| Level | Criteria |
|-------|----------|
| high | 3+ S/A sources, no contradictions |
| medium | 2+ sources OR B-tier majority |
| low | Single source OR contradictions exist |

### Step 5: Calculate Quality Metrics

```yaml
saturation: # Information completeness (0-100)
  formula: (unique_topics_covered / expected_topics) * 100
  threshold: 50

diversity: # Source variety (0-1)
  formula: unique_domains / total_sources
  threshold: 0.5

tier_quality: # Weighted average of source tiers
  formula: sum(tier_weight * source_count) / total_sources
  threshold: 0.6

evidence_depth: # Average findings per insight
  formula: total_findings / total_insights
  threshold: 3
```

### Step 6: Organize by Themes

Group insights into themes for report structure:

```yaml
themes:
  - name: "Theme Name"
    description: "What this theme covers"
    insights: ["i001", "i003", "i007"]
    primary_aspects: ["aspect_1", "aspect_2"]
```

## Output Schema

```yaml
metadata:
  session_id: string
  created_at: timestamp
  aspects_processed: number
  total_findings: number
  total_sources: number
  unique_domains: number

insights:
  - id: "i001"
    title: string
    description: string
    evidence:
      - finding_id: string
        aspect_id: string
        source_url: string
        weight: number
    confidence: high|medium|low
    type: observation|recommendation|warning

cross_aspect_patterns:
  - pattern: string
    type: recurring|contradiction|causal|gap
    aspects: string[]
    strength: number
    evidence: object[]

themes:
  - name: string
    description: string
    insights: string[]

quality_metrics:
  saturation: number
  diversity: number
  tier_quality: number
  evidence_depth: number

source_summary:
  total: number
  by_tier: {S: n, A: n, B: n, C: n, D: n}
  top_domains: string[]
```

## Example Output

```yaml
metadata:
  session_id: "research_20260130_x7k9m"
  created_at: "2026-01-30T10:30:00Z"
  aspects_processed: 5
  total_findings: 48
  total_sources: 32
  unique_domains: 24

insights:
  - id: "i001"
    title: "Hierarchical orchestration dominates production systems"
    description: "Most production multi-agent systems use hierarchical patterns with a central coordinator, despite theoretical benefits of mesh/swarm approaches."
    evidence:
      - finding_id: "arch_f003"
        aspect_id: "architecture"
        source_url: "https://..."
        weight: 0.9
      - finding_id: "tools_f007"
        aspect_id: "tools"
        source_url: "https://..."
        weight: 0.7
    confidence: high
    type: observation

cross_aspect_patterns:
  - pattern: "State management is the primary challenge across all orchestration approaches"
    type: recurring
    aspects: ["architecture", "challenges", "tools"]
    strength: 5

themes:
  - name: "Orchestration Approaches"
    description: "Different architectural patterns and their trade-offs"
    insights: ["i001", "i002", "i005"]

quality_metrics:
  saturation: 72
  diversity: 0.75
  tier_quality: 0.68
  evidence_depth: 4.2

source_summary:
  total: 32
  by_tier: {S: 3, A: 12, B: 10, C: 5, D: 2}
  top_domains: ["github.com", "arxiv.org", "langchain.com"]
```

## Quality Criteria

- [ ] All findings traced to aspects
- [ ] No hallucinated insights
- [ ] Patterns have evidence
- [ ] Themes cover all major insights
- [ ] Metrics calculated
