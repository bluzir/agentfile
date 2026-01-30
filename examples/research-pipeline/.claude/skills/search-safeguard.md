---
name: search-safeguard
type: atomic
version: v1.0
description: "Exa API wrapper with jitter, retry, error handling"
---

# Search Safeguard

## Purpose

Wrap search API calls with reliability patterns: jitter between requests, automatic retry on failure, graceful error handling.

## Configuration

```yaml
defaults:
  max_retries: 3
  base_delay_ms: 500
  jitter_range_ms: 200
  timeout_ms: 30000
  results_per_query: 8
```

## Procedure

### Step 1: Pre-Request Jitter

Before each search request:
```
delay = base_delay_ms + random(0, jitter_range_ms)
wait(delay)
```

This prevents rate limiting when running parallel requests.

### Step 2: Execute Search

```
result = mcp__exa__web_search_exa(
  query: query,
  numResults: results_per_query
)
```

### Step 3: Retry on Failure

On failure (timeout, rate limit, server error):

```
for attempt in 1..max_retries:
  delay = base_delay_ms * (2 ^ attempt) + random(0, jitter_range_ms)
  wait(delay)
  result = retry_search()
  if success:
    return result
return failure_result
```

### Step 4: Error Classification

| Error Type | Retry | Action |
|------------|-------|--------|
| Rate limit (429) | Yes | Exponential backoff |
| Server error (5xx) | Yes | Standard retry |
| Client error (4xx) | No | Log and skip |
| Timeout | Yes | Increase timeout |
| Network error | Yes | Standard retry |

### Step 5: Result Validation

After successful response:
- Check results array exists
- Filter out null/invalid results
- Return validated results

## Output

```yaml
status: success|partial|failed
queries_attempted: 3
queries_succeeded: 3
total_results: 24
errors: []
results:
  - query: "search query 1"
    count: 8
    items: [...]
```

## Error Output

```yaml
status: failed
queries_attempted: 3
queries_succeeded: 0
errors:
  - query: "search query 1"
    error: "Rate limit exceeded"
    attempts: 3
  - query: "search query 2"
    error: "Timeout after 30s"
    attempts: 3
```

## Integration

### With Researchers

```
# In researcher agent
For each query:
  Apply search-safeguard:
    - Add jitter before request
    - Execute mcp__exa__web_search_exa
    - Retry on failure
    - Collect results or log error
```

### Batch Execution

When executing multiple queries:
1. Apply jitter between each
2. Consider parallel execution with increased jitter
3. Aggregate errors at the end
4. Continue with successful results

## Best Practices

- Don't retry client errors (bad query)
- Log all errors for debugging
- Continue pipeline even if some queries fail
- Set reasonable timeouts
- Use exponential backoff for rate limits
