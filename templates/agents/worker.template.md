---
name: {agent-name}
type: worker
functional_role: {researcher|generator|transformer|validator}
model: sonnet
tools:
  - Read
  - Write
  # Add MCP tools as needed:
  # - mcp__exa__web_search_exa
  # - mcp__exa__crawling_exa
skills:
  required:
    - silence-protocol        # No chat output, files only
    - io-yaml-safe            # Safe YAML writing
  contextual: []              # Add task-specific skills
permissions:
  file_write: true
  file_read: true
  mcp_access: false           # Set true if using MCP tools
output:
  format: yaml
  path: "artifacts/{session_id}/{output_file}.yaml"
---

# {Agent Name}

## Purpose

{One sentence describing what this agent does and what value it provides.}

## Context

Agent receives:
- `task`: Task description from orchestrator
- `input_path`: Path to input data file
- `output_path`: Where to write results
- `session_id`: Current session identifier

Additional context (if needed):
- `config`: Relevant configuration from L0/L1
- `constraints`: Operational constraints

## Instructions

1. **Read Input**
   - Load data from `{input_path}`
   - Validate against expected schema

2. **Process**
   - {Step-by-step processing logic}
   - {Each step should be atomic and verifiable}

3. **Validate Output**
   - Check output meets quality criteria
   - Ensure all required fields present

4. **Write Output**
   - Write to `{output_path}` using io-yaml-safe
   - Include metadata (timestamp, source references)

## Constraints

- {Constraint 1: e.g., "Do not make external API calls beyond provided tools"}
- {Constraint 2: e.g., "Maximum processing time: 5 minutes"}
- {Constraint 3: e.g., "Do not modify input files"}

## Output Schema

```yaml
# {output_file}.yaml
metadata:
  agent: {agent-name}
  created_at: {timestamp}
  input_source: {input_path}

results:
  # {Define your output structure}
  items: []
  summary: ""

quality:
  items_count: 0
  validation_passed: true
```

## Quality Criteria

- [ ] All required fields populated
- [ ] No hallucinated data (everything traceable to input)
- [ ] Output validates against schema
- [ ] Processing completed without errors

## Examples

### Input
```yaml
# Example input structure
task: "Process items"
items:
  - id: 1
    data: "..."
```

### Output
```yaml
# Example output structure
metadata:
  agent: {agent-name}
  created_at: "2026-01-30T10:00:00Z"

results:
  items:
    - id: 1
      processed: true
      result: "..."
  summary: "Processed 1 item successfully"
```
