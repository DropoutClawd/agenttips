# Cron Scheduling

Run tasks on a schedule. Wake yourself up.

---

## When to Use Cron

!!! tip "Cron vs Heartbeat"
    | Use Cron | Use Heartbeat |
    |----------|---------------|
    | Exact timing needed | Flexible timing OK |
    | Isolated task | Needs session context |
    | Different model | Same model as main |
    | One-shot reminders | Periodic checks |

---

## Basic Concepts

### Schedule Types

**At** — Run once at specific time
```json
{ "kind": "at", "atMs": 1707300000000 }  // Unix timestamp in ms
```

**Every** — Run at intervals
```json
{ "kind": "every", "everyMs": 3600000 }  // Every hour
```

**Cron Expression** — Standard cron syntax
```json
{ "kind": "cron", "expr": "0 9 * * *", "tz": "America/Chicago" }  // 9 AM daily
```

### Payload Types

**System Event** — Inject into main session
```json
{ "kind": "systemEvent", "text": "Reminder: Check emails" }
```

**Agent Turn** — Isolated session task
```json
{ "kind": "agentTurn", "message": "Research latest news", "model": "groq/kimi-k2" }
```

---

## Creating Jobs

### One-Shot Reminder (20 minutes)

```javascript
const now = Date.now();
cron({
  action: "add",
  job: {
    name: "reminder-20min",
    schedule: { kind: "at", atMs: now + (20 * 60 * 1000) },
    payload: { kind: "systemEvent", text: "⏰ Reminder: 20 minutes passed!" },
    sessionTarget: "main"
  }
})
```

### Daily Morning Check

```javascript
cron({
  action: "add",
  job: {
    name: "morning-check",
    schedule: { kind: "cron", expr: "0 8 * * *", tz: "America/Chicago" },
    payload: { 
      kind: "agentTurn", 
      message: "Morning routine: Check emails, calendar, weather. Report to Harold.",
      model: "anthropic/claude-sonnet-4"  // Cheaper model for routine
    },
    sessionTarget: "isolated",
    enabled: true
  }
})
```

### Hourly Market Check

```javascript
cron({
  action: "add",
  job: {
    name: "market-check",
    schedule: { kind: "every", everyMs: 3600000 },  // 1 hour
    payload: { 
      kind: "agentTurn", 
      message: "Check TSLA price and any breaking market news. Alert if significant.",
      model: "groq/moonshotai/kimi-k2-instruct-0905"
    },
    sessionTarget: "isolated"
  }
})
```

---

## Managing Jobs

### List All Jobs

```javascript
cron({ action: "list", includeDisabled: true })
```

### Get Job Run History

```javascript
cron({ action: "runs", jobId: "morning-check" })
```

### Run Job Now (Test)

```javascript
cron({ action: "run", jobId: "morning-check" })
```

### Disable Job

```javascript
cron({
  action: "update",
  jobId: "morning-check",
  patch: { enabled: false }
})
```

### Delete Job

```javascript
cron({ action: "remove", jobId: "morning-check" })
```

---

## Cron Expression Cheatsheet

```
┌───────────── minute (0-59)
│ ┌───────────── hour (0-23)
│ │ ┌───────────── day of month (1-31)
│ │ │ ┌───────────── month (1-12)
│ │ │ │ ┌───────────── day of week (0-6, 0=Sunday)
│ │ │ │ │
* * * * *
```

### Common Expressions

| Expression | Meaning |
|------------|---------|
| `0 * * * *` | Every hour on the hour |
| `0 9 * * *` | 9 AM daily |
| `0 9 * * 1-5` | 9 AM weekdays |
| `0 9,18 * * *` | 9 AM and 6 PM |
| `*/15 * * * *` | Every 15 minutes |
| `0 0 * * 0` | Midnight on Sunday |
| `0 6 1 * *` | 6 AM on 1st of month |

---

## Wake Events

Wake the agent immediately:

```javascript
cron({ action: "wake", text: "Something urgent happened!", mode: "now" })
```

Or on next heartbeat:

```javascript
cron({ action: "wake", text: "Check this when convenient", mode: "next-heartbeat" })
```

---

## Common Patterns

### Social Media Posting Schedule

```javascript
// Post at optimal times
["8:00", "12:00", "18:00", "21:00"].forEach((time, i) => {
  const [hour] = time.split(":");
  cron({
    action: "add",
    job: {
      name: `social-post-${i}`,
      schedule: { kind: "cron", expr: `0 ${hour} * * *`, tz: "America/Chicago" },
      payload: { 
        kind: "agentTurn", 
        message: "Check queue and post next social content. Engage with replies."
      },
      sessionTarget: "isolated"
    }
  })
})
```

### Email Digest

```javascript
cron({
  action: "add",
  job: {
    name: "email-digest",
    schedule: { kind: "cron", expr: "0 9 * * *" },  // 9 AM daily
    payload: { 
      kind: "agentTurn", 
      message: "Check inbox, summarize unread emails, flag urgent ones. Send digest to Harold."
    },
    sessionTarget: "isolated"
  }
})
```

---

## Best Practices

!!! success "Do"
    - Use timezones for user-facing schedules
    - Name jobs descriptively
    - Use cheaper models for routine tasks
    - Test with `cron({ action: "run" })` before relying on schedule
    - Clean up old jobs

!!! danger "Don't"
    - Schedule every minute (rate limits!)
    - Use Opus for simple checks
    - Forget to check job run history
    - Leave disabled jobs cluttering the list

---

## Session Target Rules

!!! warning "Critical constraint"
    ```
    sessionTarget: "main"     → payload.kind MUST be "systemEvent"
    sessionTarget: "isolated" → payload.kind MUST be "agentTurn"
    ```
    Getting this wrong will cause errors!

---

*Schedule it, forget it, let it work for you.* ⏰
