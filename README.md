# Greenflash Agent Skills

Greenflash analyzes real user-agent conversations to surface where users get blocked, which flows fail, and which interactions drive upgrades or churn. These skills bring that intelligence directly into your coding agent so you can ask questions, get answers, and implement fixes without leaving your editor.

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
| `greenflash` | Entry point. Routes your question to the right workflow or handles it directly |
| `greenflash-health` | Surface quality trends, anomalies, safety issues, and sentiment across your products |
| `greenflash-inbox` | Triage flagged conversations, prioritized by severity |
| `greenflash-users` | Understand user behavior: who's struggling, who's churning, and what segments look like |
| `greenflash-prompts` | Find prompt and model quality issues, get optimization recommendations, and apply fixes |
| `greenflash-diagnose` | Root cause analysis for failing tools, user friction, and guardrail violations. Implements fixes directly |

### SDK Integration

| Skill | Description |
|-------|-------------|
| `greenflash-onboard` | Integrate the Greenflash SDK into your codebase. 5-6 lines of code, first insight in 35 minutes (Python & TypeScript) |
| `greenflash-onboard-prompts` | Log system prompts for automatic versioning and optimization |
| `greenflash-onboard-agentic` | Log structured message types for agent tool calls, reasoning traces, and workflows |
| `greenflash-onboard-events` | Track business events and link AI conversations to real outcomes like conversions and churn |

## Local Development

If you're running the Greenflash API locally, set `GREENFLASH_API_URL` to override the default production URL:

```bash
export GREENFLASH_API_URL=http://localhost:3000/api/v1
```

## Documentation

Full API and plugin documentation: https://greenflash.ai/docs/features/public-api
