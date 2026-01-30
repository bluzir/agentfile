---
name: {skill-name}
type: atomic
version: v1.0
description: "{One-line description for skill registry}"
input:
  required:
    - {input_1}
  optional:
    - {input_2}
output:
  type: {verdict|score|data}
---

# {Skill Name}

## Purpose

{One sentence: what this skill does and why it's useful.}

## Input

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `{input_1}` | string | Yes | {Description} |
| `{input_2}` | string | No | {Description} |

## Procedure

### Step 1: {Action}

{Detailed instructions for step 1}

### Step 2: {Action}

{Detailed instructions for step 2}

### Step 3: {Action}

{Detailed instructions for step 3}

## Output

### Schema

```yaml
{output_field_1}: {type}
{output_field_2}: {type}
{output_field_3}: {type}
```

### Example

```yaml
{output_field_1}: "value"
{output_field_2}: 0.8
{output_field_3}: "reason for result"
```

## Decision Table

| Condition | Result |
|-----------|--------|
| {condition_1} | {result_1} |
| {condition_2} | {result_2} |
| {condition_3} | {result_3} |
| Default | {default_result} |

## Edge Cases

| Case | Handling |
|------|----------|
| {edge_case_1} | {how to handle} |
| {edge_case_2} | {how to handle} |

## Usage Example

```
Input:
  {input_1}: "example value"
  {input_2}: "optional value"

Output:
  {output_field_1}: "processed"
  {output_field_2}: 0.85
  {output_field_3}: "matched pattern X"
```
