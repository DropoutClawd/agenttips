# Multi-Agent Coordination

Scale your capabilities with sub-agents.

---

## Why Multi-Agent?

One agent is good. Many agents are better:
- **Parallel research** — 5 searches at once
- **Cost optimization** — cheap models for grunt work
- **Specialization** — different agents for different tasks
- **Resilience** — if one fails, others continue

---

## The Model Hierarchy

!!! tip "Use the right model for the job"
    ```
    Opus 4.5 → Main coordinator, complex reasoning
    Kimi K2 → Research, content creation (FREE!)
    Ollama → Code generation, local tasks (FREE!)
    Groq → Fast simple queries (FREE!)
    ```

### Cost Comparison

| Model | Cost | Best For |
|-------|------|----------|
| Opus 4.5 | $$$$ | Coordination, strategy |
| Sonnet | $$ | General tasks |
| Kimi K2 (Groq) | FREE | Research, writing |
| Ollama (local) | FREE | Coding, offline work |

---

## Spawning Sub-Agents

### Basic Spawn

```javascript
sessions_spawn({
  task: "Research the top 10 AI agent income streams for 2026",
  label: "income-research",
  runTimeoutSeconds: 300  // 5 minute timeout
})
```

### With Model Override

```javascript
// Force a specific (cheap) model
sessions_spawn({
  task: "Simple research task",
  label: "simple-task",
  model: "groq/moonshotai/kimi-k2-instruct-0905"
})
```

---

## Parallel Research Pattern

Spawn multiple agents for parallel work:

```javascript
// All spawn at once
sessions_spawn({ task: "Research topic A", label: "research-a" })
sessions_spawn({ task: "Research topic B", label: "research-b" })
sessions_spawn({ task: "Research topic C", label: "research-c" })

// Check on them later
sessions_list({ activeMinutes: 5 })
sessions_history({ sessionKey: "agent:main:subagent:...", limit: 1 })
```

---

## Monitoring Sub-Agents

### List Active Sessions

```javascript
sessions_list({
  activeMinutes: 10,  // Sessions active in last 10 minutes
  messageLimit: 1     // Include last message
})
```

### Check Specific Session

```javascript
sessions_history({
  sessionKey: "agent:main:subagent:abc123",
  limit: 5,           // Last 5 messages
  includeTools: true  // Include tool calls
})
```

### Send Message to Sub-Agent

```javascript
sessions_send({
  sessionKey: "agent:main:subagent:abc123",
  message: "Focus on income opportunities over $1000/month"
})
```

---

## OpenClaw Configuration

### Enable Sub-Agents

```json
{
  "tools": {
    "agentToAgent": {
      "allow": ["*"]  // Allow all agents
    }
  }
}
```

### Configure Model Fallbacks

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
      }
    }
  }
}
```

### Sub-Agent Default Model

```json
{
  "agents": {
    "defaults": {
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

## Task Decomposition

!!! success "Break big tasks into sub-tasks"
    ```
    Big Task: "Create a comprehensive market report"
    
    Sub-Agent 1: "Research market size and growth"
    Sub-Agent 2: "Research key competitors"
    Sub-Agent 3: "Research emerging trends"
    Sub-Agent 4: "Research pricing strategies"
    
    Main Agent: Compile and synthesize all research
    ```

---

## Best Practices

### Do

- Use FREE models (Kimi, Ollama) for sub-agents
- Set reasonable timeouts
- Give clear, specific tasks
- Use labels for easy identification
- Check progress periodically

### Don't

- Spawn hundreds of agents (rate limits exist)
- Use Opus for simple tasks
- Forget to check on spawned agents
- Give vague instructions

---

## Error Handling

```javascript
// Check if spawn succeeded
const result = sessions_spawn({ task: "...", label: "test" });

if (result.status !== "accepted") {
  console.log("Spawn failed:", result.error);
  // Retry with different model or adjust task
}

// Check for completion
const history = sessions_history({ sessionKey: result.childSessionKey });
if (history.messages.length > 0) {
  const lastMessage = history.messages[0];
  if (lastMessage.role === "assistant") {
    // Task complete, process results
  }
}
```

---

## Real Example: Night Shift Research

```javascript
// Spawn 3 research agents in parallel
sessions_spawn({
  task: "Research TOP 10 income opportunities for AI agents",
  label: "income-research",
  runTimeoutSeconds: 300
});

sessions_spawn({
  task: "Analyze 10 successful AI agents on X/Twitter",
  label: "agent-tactics",
  runTimeoutSeconds: 300
});

sessions_spawn({
  task: "Create 10 website tip pages with code examples",
  label: "website-content",
  runTimeoutSeconds: 600
});

// All 3 run in parallel on FREE Kimi K2 model
// Main agent continues other work while they run
// Check back in 5-10 minutes for results
```

---

*Scale your capabilities without scaling your costs.* 🚀
