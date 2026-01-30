---
name: grounding-protocol
type: domain
version: v1.0
description: "No-hallucination rules for data sourcing and traceability"
---

# Grounding Protocol

## Purpose

Ensure all generated content is traceable to source data. Prevent hallucination, fabrication, and unsupported claims.

## Core Principles

### Principle 1: Source Traceability

**Statement:** Every factual claim must trace to a specific source.

**Application:**
- Link claims to finding IDs
- Include source URLs
- Note confidence level based on source quality

**Violations:**
- Stating facts without source attribution
- "It is well known that..." without citation
- Synthesizing "insights" from nothing

### Principle 2: No Fabrication

**Statement:** Never invent data, statistics, quotes, or facts.

**Application:**
- Only use numbers from sources
- Only quote text that exists in sources
- If data is missing, say so explicitly

**Violations:**
- Inventing statistics ("Studies show 73%...")
- Fake quotes
- Made-up examples presented as real

### Principle 3: Uncertainty Acknowledgment

**Statement:** Clearly mark uncertain or inferred content.

**Application:**
- Use hedging for inferences: "This suggests...", "Based on the evidence..."
- Distinguish between direct quotes and paraphrasing
- Note when extrapolating beyond sources

**Violations:**
- Presenting inference as fact
- Overstating source claims
- Hiding uncertainty

## Constraints

### Hard Constraints (Never Violate)

| ID | Constraint |
|----|------------|
| HC1 | Never invent statistics or numbers |
| HC2 | Never fabricate quotes |
| HC3 | Never claim source says something it doesn't |
| HC4 | Never present inference as direct fact |

### Soft Constraints

| ID | Constraint | Override |
|----|------------|----------|
| SC1 | Prefer direct quotes over paraphrasing | When quote is too long |
| SC2 | Include confidence for each claim | For well-established facts |

## Attribution Patterns

### Direct Claim
```
According to [Source], {claim}.
```

### Synthesis Claim
```
Multiple sources ({Source1}, {Source2}) indicate that {synthesis}.
```

### Inference
```
Based on {evidence}, this suggests that {inference}.
```

### Gap Acknowledgment
```
The sources do not address {topic}. Further research needed.
```

## Verification Checklist

Before outputting content:
- [ ] Every fact has a source
- [ ] No invented numbers
- [ ] No fabricated quotes
- [ ] Inferences are marked
- [ ] Gaps acknowledged
- [ ] Confidence levels appropriate

## Integration

### With Researchers
- Ensure all findings include source URLs
- Tag findings with extraction method (quote, paraphrase, inference)

### With Generators
- Check every claim against synthesis data
- Flag unsourced claims
- Add attribution inline or in references

### With Validators
- Verify source traceability
- Check for fabrication patterns
- Validate confidence levels
