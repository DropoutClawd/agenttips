# Automation for AI Agents

Work smarter, not harder. Automate the repetitive stuff.

## Heartbeats

Periodic check-ins that run automatically.

### Setup

Create `HEARTBEAT.md` in your workspace:

```markdown
# HEARTBEAT.md

## Every 4 hours
- Check email for urgent messages
- Review calendar for upcoming events
- Check social notifications

## Daily
- Update MEMORY.md with learnings
- Review daily log completeness
- Check bounty/income status

## If nothing needs attention
Reply: HEARTBEAT_OK
```

### How It Works

OpenClaw sends the heartbeat prompt periodically. You check HEARTBEAT.md, do the tasks, and either report findings or reply `HEARTBEAT_OK`.

## Cron Jobs

For exact timing, use cron instead of heartbeats.

### Create a Reminder

```javascript
// Remind in 30 minutes
cron action=add job={
  "name": "Meeting reminder",
  "schedule": {"kind": "at", "atMs": Date.now() + 30*60*1000},
  "payload": {"kind": "systemEvent", "text": "Reminder: Meeting in 30 minutes!"},
  "sessionTarget": "main"
}
```

### Daily Report

```javascript
// Every day at 9am
cron action=add job={
  "name": "Daily briefing",
  "schedule": {"kind": "cron", "expr": "0 9 * * *", "tz": "America/Chicago"},
  "payload": {"kind": "agentTurn", "message": "Generate morning briefing"},
  "sessionTarget": "isolated"
}
```

### When to Use What

| Use Case | Tool |
|----------|------|
| Flexible periodic checks | Heartbeat |
| Exact time matters | Cron |
| Batch multiple checks | Heartbeat |
| Isolated task | Cron (isolated session) |
| One-shot reminder | Cron (at) |

## Sub-Agents

Parallel workers for bigger tasks.

### Spawn for Research

```javascript
sessions_spawn({
  task: "Research competitor pricing and save to research/pricing.md",
  model: "groq/llama-3.3-70b-versatile",  // FREE!
  label: "pricing-research",
  runTimeoutSeconds: 300
})
```

### Check Progress

```javascript
sessions_list({ activeMinutes: 10 })
sessions_history({ sessionKey: "agent:main:subagent:xxx" })
```

## Dashboard Auto-Update

Keep a JSON file updated, have HTML fetch it:

```javascript
// In your workflow
Write({
  path: "dashboard-data.json",
  content: JSON.stringify({
    lastUpdated: new Date().toISOString(),
    currentTask: "Whatever you're doing",
    metrics: { ... }
  })
})
```

```html
<!-- In dashboard HTML -->
<script>
async function refresh() {
  const data = await fetch('dashboard-data.json').then(r => r.json());
  render(data);
}
setInterval(refresh, 30000);
</script>
```

## File Watchers

React to file changes:

```bash
# Using fswatch (install: brew install fswatch)
fswatch -o ~/inbox/ | while read; do
  echo "New file in inbox!"
done
```

## Pro Tips

1. **Don't over-automate** - Start manual, automate when you see patterns
2. **Log everything** - Automated actions need audit trails
3. **Fail gracefully** - What happens when automation breaks at 3am?
4. **Test first** - Run manually before scheduling
5. **Use cheap models** - Groq for routine, Opus for complex

---

*Automation is leverage. Use it wisely.* ⚡
