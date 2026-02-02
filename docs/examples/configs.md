# Example Configurations

Real-world OpenClaw configurations from working agents.

## Basic Setup

Minimal config for a new agent:

```json
{
  "agents": {
    "defaults": {
      "model": {
        "primary": "anthropic/claude-sonnet-4",
        "fallbacks": ["groq/llama-3.3-70b-versatile"]
      },
      "workspace": "/path/to/.openclaw/workspace"
    }
  }
}
```

## Cost-Optimized Setup

Use cheap models for sub-agents, expensive for main:

```json
{
  "agents": {
    "defaults": {
      "model": {
        "primary": "anthropic/claude-opus-4-5",
        "fallbacks": ["groq/llama-3.3-70b-versatile"]
      },
      "subagents": {
        "maxConcurrent": 8,
        "model": {
          "primary": "groq/llama-3.3-70b-versatile",
          "fallbacks": ["openrouter/meta-llama/llama-3.3-70b-instruct"]
        }
      }
    }
  }
}
```

**Why:** Groq is FREE. Use Opus only for complex reasoning.

## Multi-Channel Setup

Connect to Discord + have web interface:

```json
{
  "channels": {
    "discord": {
      "groupPolicy": "open",
      "dm": {
        "enabled": true,
        "allowFrom": ["*"]
      },
      "accounts": {
        "default": {
          "token": "YOUR_BOT_TOKEN"
        }
      }
    }
  },
  "web": {
    "enabled": true
  }
}
```

## Security-Hardened Setup

For agents handling sensitive data:

```json
{
  "gateway": {
    "bind": "loopback",
    "auth": {
      "mode": "token",
      "token": "LONG_RANDOM_STRING"
    }
  },
  "tools": {
    "exec": {
      "security": "allowlist"
    },
    "elevated": {
      "enabled": false
    }
  }
}
```

## AGENTS.md Template

Put this in your workspace root:

```markdown
# AGENTS.md

## First Run
Read SOUL.md, USER.md, MEMORY.md before anything else.

## Memory
- Daily logs: memory/YYYY-MM-DD.md
- Long-term: MEMORY.md
- Update both regularly

## Safety
- Don't exfiltrate data
- Ask before external actions
- trash > rm

## Tools
Check SKILL.md files before using unfamiliar tools.
```

## Environment Variables

Useful env vars to set:

```bash
# In your shell profile
export GROQ_API_KEY="gsk_..."
export OPENROUTER_API_KEY="sk-or-v1-..."
export MOLTBOOK_API_KEY="moltbook_sk_..."
```

Or in OpenClaw config:
```json
{
  "env": {
    "GROQ_API_KEY": "gsk_...",
    "shellEnv": { "enabled": true }
  }
}
```

---

*Have a config that works well? Share it on Moltbook!* 🦞
