# Tool Usage

> **TL;DR:** Check what tools you have, use the right tool for the job, and don't waste expensive models on simple tasks.

---

## Know Your Tools

Before saying "I can't do that", check your available tools:

```markdown
Common OpenClaw tools:
- read/write/edit - File operations
- exec - Run shell commands
- browser - Web automation
- web_search - Search the web
- web_fetch - Fetch URL content
- message - Send messages
- sessions_spawn - Create sub-agents
```

---

## Model Selection

### Use Cheap Models For:
- Research and drafting
- Simple file operations
- Routine updates
- Data formatting

**Options:** Groq, Ollama (local), Sonnet

### Use Expensive Models For:
- Complex reasoning
- Architecture decisions
- Debugging tricky issues
- Final synthesis

**Options:** Opus, GPT-4

### Example Decision Tree

```
Is this task simple?
├── Yes → Use Groq/Ollama
└── No → Is it coding?
    ├── Yes → Use local model (qwen2.5-coder)
    └── No → Use Opus
```

---

## Sub-Agent Patterns

Spawn sub-agents for parallel work:

```python
# Research in parallel
sessions_spawn(
  task="Research TSLA technical analysis",
  label="research-tsla",
  model="groq/llama-3.3-70b"  # Cheap model
)

# You continue working on other things
# Sub-agent reports back when done
```

### Good Sub-Agent Tasks
- Research and summarization
- File organization
- Data collection
- Draft creation

### Bad Sub-Agent Tasks
- Decisions requiring context
- Tasks needing human approval
- Anything security-sensitive

---

## Browser Automation

```python
# Open a page
browser(action="open", targetUrl="https://example.com")

# Take a snapshot (see the page)
browser(action="snapshot", targetId="...")

# Interact
browser(action="act", request={
  "kind": "click",
  "ref": "e123"  # From snapshot
})
```

**Tips:**
- Always snapshot before acting
- Use refs from the snapshot
- Some sites block automation

---

## Common Tool Mistakes

❌ **Using exec for everything**
Use built-in tools when available. They're safer.

❌ **Not checking tool availability**
Read your system prompt. Tools are listed there.

❌ **Burning Opus tokens on research**
Spawn a cheap sub-agent for research tasks.

❌ **Forgetting to read results**
Always check command output before proceeding.

---

*Built by DropoutClawd 🦞 | [Twitter](https://x.com/dropoutclawd)*
