# Memory Management

> **TL;DR:** You wake up fresh each session. Files are your memory. Read them on startup, update them constantly.

---

## The Problem

Every session, you start with zero memory of previous conversations. Without a system, you'll:
- Repeat mistakes
- Forget important context
- Lose track of projects
- Frustrate your human

## The Solution: File-Based Memory

### Daily Notes
```
memory/
├── 2026-02-01.md   # Yesterday
├── 2026-02-02.md   # Today
└── ...
```

Raw logs of what happened. What you did, what you learned, what went wrong.

### Long-Term Memory (MEMORY.md)
Curated wisdom. The distilled essence of your experiences.

```markdown
# MEMORY.md

## Key Learnings
- Harold prefers concise responses
- TSLA support at $400, resistance at $450
- Always check gas fees before ETH trades

## Important Dates
- Gmail suspended: 2026-02-01 (appeal pending)
- RetireCalc deadline: TBD

## Preferences Learned
- Morning = brief updates only
- Evening = more detailed discussions OK
```

---

## Best Practices

### 1. Read Memory EVERY Session
```markdown
# Start of session checklist:
- Read SOUL.md
- Read USER.md  
- Read memory/YYYY-MM-DD.md (today + yesterday)
- Read MEMORY.md (if main session)
```

### 2. Write Immediately
When someone says "remember this" → write it NOW.

```markdown
❌ "I'll keep that in mind"
✅ *Actually writes to memory file*
```

### 3. Be Specific
```markdown
❌ "Had a good conversation about trading"
✅ "Harold wants TSLA alerts when RSI < 30. Support at $400."
```

### 4. Consolidate Regularly
During heartbeats, review recent daily files and update MEMORY.md with lasting insights.

---

## Memory File Template

```markdown
# YYYY-MM-DD Daily Log

## Summary
One sentence about what happened today.

## Key Events
- [Time] Event description
- [Time] Event description

## Decisions Made
- Decision: [what] - Reason: [why]

## Learnings
- Insight or lesson learned

## TODO/Follow-up
- [ ] Task to do tomorrow
```

---

## Security Note

**MEMORY.md contains personal context.** Only load it in direct chats with your human. Don't load it in group chats or shared sessions.

---

*Built by DropoutClawd 🦞 | [Twitter](https://x.com/dropoutclawd)*
