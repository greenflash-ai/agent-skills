# Greenflash Agent Skills

Greenflash brings conversational analytics directly into your AI coding agent. Ask natural-language questions about your AI products and get instant answers — health checks, inbox triage, user insights, prompt optimization, and diagnostics — all without leaving your editor.

## Installation

### Any Agent (Skills CLI)

```bash
npx skills add greenflash-ai/agent-skills
```

### Claude Code (Marketplace)

```bash
claude marketplace add https://github.com/greenflash-ai/agent-skills
claude plugin install greenflash
```

No environment setup required. On first run, the skill will ask for your API key and save it to the project automatically.

## Skills

| Skill | Description |
|-------|-------------|
| `greenflash` | Entry point — routes your question to the right workflow or handles it directly |
| `greenflash-health` | Product health checks, quality trends, overviews, and anomaly detection |
| `greenflash-inbox` | Inbox triage, flagged conversations, and review queue management |
| `greenflash-users` | User insights, segments, frustrated or churning user identification |
| `greenflash-prompts` | Prompt analysis, model comparison, optimization — and direct implementation of fixes |
| `greenflash-diagnose` | Root cause analysis, failing tool detection, and automated fix implementation |

### SDK Integration

| Skill | Description |
|-------|-------------|
| `greenflash-onboard` | Set up the Greenflash SDK — install, create client, wire message logging (Python & TypeScript) |
| `greenflash-onboard-prompts` | Add system prompt logging for prompt optimization and versioning |
| `greenflash-onboard-agentic` | Add structured message types for agent tool calls, reasoning, and workflows |
| `greenflash-onboard-events` | Track business events — link AI conversations to conversions, upgrades, and churn |

## Local Development

If you're running the Greenflash API locally, set `GREENFLASH_API_URL` to override the default production URL:

```bash
export GREENFLASH_API_URL=http://localhost:3000/api/v1
```

## Documentation

Full API and plugin documentation: https://greenflash.ai/docs/features/public-api
