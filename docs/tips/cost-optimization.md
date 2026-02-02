# Cost Optimization

How to maximize output while minimizing spend.

---

## The Problem

AI APIs cost money. A lot of it.

| Model | Input | Output | 1M tokens |
|-------|-------|--------|-----------|
| Opus 4.5 | $0.015 | $0.075 | ~$45 |
| Sonnet 4 | $0.003 | $0.015 | ~$9 |
| Groq (Kimi K2) | FREE | FREE | $0 |
| Local Ollama | FREE | FREE | $0 |

**The math is clear:** Use expensive models sparingly.

---

## Strategy 1: Model Routing

!!! success "Route by complexity"
    ```
    Complex reasoning → Opus
    Simple tasks → Groq/Ollama
    Code generation → Local model
    Research → Kimi K2 (FREE, 262K context!)
    ```

### OpenClaw Config

```json
{
  "agents": {
    "defaults": {
      "model": {
        "primary": "anthropic/claude-opus-4-5",
        "fallbacks": [
          "groq/moonshotai/kimi-k2-instruct-0905",
          "ollama/qwen2.5-coder:7b"
        ]
      },
      "subagents": {
        "model": {
          "primary": "groq/moonshotai/kimi-k2-instruct-0905"
        }
      }
    }
  }
}
```

---

## Strategy 2: Sub-Agent Delegation

!!! tip "Offload to free models"
    ```python
    # Main agent (Opus): Coordinates
    # Sub-agents (FREE): Do the work
    
    sessions_spawn({
      task: "Research competitor pricing",
      model: "groq/moonshotai/kimi-k2-instruct-0905",  # FREE!
      label: "pricing-research"
    })
    ```

**Result:** Complex coordination on Opus, grunt work on Kimi (FREE).

---

## Strategy 3: Context Management

!!! warning "Long context = expensive"
    Every token in your context costs money on every response.

**Reduce context:**
- Summarize long conversations
- Offload reference data to files
- Use `memory_search` instead of loading everything
- Enable compaction (`mode: "safeguard"`)

```json
{
  "agents": {
    "defaults": {
      "compaction": {
        "mode": "safeguard"
      }
    }
  }
}
```

---

## Strategy 4: Caching

!!! info "Don't repeat expensive calls"
    ```python
    # Check cache first
    cache_file = f"cache/{query_hash}.json"
    if file_exists(cache_file):
        return read_json(cache_file)
    
    # Only call API if not cached
    result = expensive_api_call()
    write_json(cache_file, result)
    return result
    ```

---

## Strategy 5: Batch Operations

!!! tip "One big call > many small calls"
    ```
    ❌ 10 separate API calls for 10 items
    ✅ 1 API call processing all 10 items
    ```

Example:
```markdown
# Instead of:
"Analyze item 1" → response
"Analyze item 2" → response
...

# Do this:
"Analyze items 1-10 and return results as JSON array"
→ one response with all results
```

---

## The Free Tier Playbook

### Groq (FREE)
- Kimi K2: 262K context, great quality
- Llama 3.3 70B: Fast, capable
- Rate limits exist but generous

### Ollama (FREE, Local)
- qwen2.5-coder: Great for code
- llama3.3: General purpose
- No rate limits, no API costs

### OpenRouter (Cheap)
- Some free models available
- Pay-per-use for others
- Good fallback option

---

## Monitoring Costs

### Track Token Usage
```python
# After each response, log:
print(f"Tokens: {usage.input + usage.output}")
print(f"Cost: ${usage.cost.total:.4f}")
```

### Daily Budget Alerts
```python
# In your heartbeat:
if daily_spend > budget_limit:
    notify_human("Budget exceeded!")
    switch_to_free_models()
```

---

## Quick Wins

| Action | Savings |
|--------|---------|
| Use Groq for sub-agents | ~$50/day |
| Enable compaction | ~20% context |
| Batch API calls | ~30% fewer calls |
| Cache common queries | Variable |
| Use local models for code | ~$20/day |

---

## Anti-Patterns

!!! danger "Expensive mistakes"
    
    - Running Opus for simple tasks
    - Huge prompts with redundant context
    - No fallback chain configured
    - Calling APIs in loops without batching
    - Not tracking costs until bill arrives

---

*Every dollar saved is a dollar earned. Optimize relentlessly.* 💰
