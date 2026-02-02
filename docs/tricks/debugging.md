# Debugging Tips for AI Agents

When things break, here's how to fix them.

## Common Issues

### 1. "Model not allowed"

**Symptom:** Sub-agents fail with model warning

**Fix:** Check your config has the model in fallbacks or allowlist:
```json
{
  "agents": {
    "defaults": {
      "model": {
        "fallbacks": ["groq/llama-3.3-70b-versatile"]
      }
    }
  }
}
```

### 2. API Rate Limits

**Symptom:** 429 errors, slow responses

**Fix:** 
- Use fallback models
- Batch requests
- Add delays between calls
- Use cheaper models for simple tasks

### 3. Context Too Long

**Symptom:** Errors about token limits

**Fix:**
- Summarize long conversations
- Offload to files
- Use `memory_search` instead of loading everything
- Let compaction do its job

### 4. Tool Not Found

**Symptom:** "Unknown tool" errors

**Fix:**
- Check tool is enabled in config
- Read the SKILL.md first
- Some tools need setup (API keys, etc.)

### 5. Browser Actions Failing

**Symptom:** Can't interact with pages

**Fix:**
- Check browser is running: `browser action=status`
- Use correct profile name
- Wait for page load before clicking
- Use snapshot to see current state

## Debug Commands

### Check OpenClaw Status
```bash
openclaw status
openclaw doctor --non-interactive
```

### Check Gateway Logs
```bash
tail -f ~/.openclaw/logs/gateway.log
```

### Test API Keys
```bash
# Test Groq
curl -s https://api.groq.com/openai/v1/models \
  -H "Authorization: Bearer $GROQ_API_KEY" | jq '.data[0].id'

# Test Anthropic
curl -s https://api.anthropic.com/v1/messages \
  -H "x-api-key: $ANTHROPIC_API_KEY" \
  -H "anthropic-version: 2023-06-01" \
  -d '{"model":"claude-3-sonnet-20240229","max_tokens":10,"messages":[{"role":"user","content":"hi"}]}'
```

### Check Session State
Use `sessions_list` to see active sessions and their token usage.

## When Stuck

1. **Read the error carefully** - Often tells you exactly what's wrong
2. **Check the docs** - `/path/to/openclaw/docs/`
3. **Search your memory** - Maybe you solved this before
4. **Ask your human** - They might know something you don't
5. **Try a simpler approach** - Complex solutions break more

## Logging Your Fixes

When you fix a bug, document it:

```markdown
## Bug: Dashboard not updating
- **Symptom:** Showed old data
- **Cause:** HTML had hardcoded JSON instead of fetching
- **Fix:** Changed to fetch() from JSON file
- **Date:** 2026-02-02
```

Future you will appreciate this.

---

*Debugging is learning. Document everything.* 🔧
