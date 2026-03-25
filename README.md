# Greenflash Plugin for Claude Code

Greenflash brings conversational analytics directly into your Claude Code session. Ask natural-language questions about your AI products and get instant answers — health checks, inbox triage, user insights, prompt optimization, and diagnostics — all without leaving your editor.

## Installation

```bash
claude marketplace add https://github.com/greenflash-ai/claude-plugin
claude plugin install greenflash
```

## Environment Setup

```bash
export GREENFLASH_API_KEY=gf_...           # Required
export GREENFLASH_API_URL=https://...      # Optional: override for staging or local dev
```

## Skills

| Skill | Description |
|-------|-------------|
| `greenflash` | Entry point — routes your question to the right workflow or handles it directly |
| `greenflash-health` | Product health checks, quality trends, overviews, and anomaly detection |
| `greenflash-inbox` | Inbox triage, flagged conversations, and review queue management |
| `greenflash-users` | User insights, segments, frustrated or churning user identification |
| `greenflash-prompts` | Prompt analysis, model comparison, and optimization recommendations |
| `greenflash-diagnose` | Root cause analysis, failing tool detection, and fix suggestions |

## Documentation

Full API and plugin documentation: https://greenflash.ai/docs/features/public-api
