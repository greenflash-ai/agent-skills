# Greenflash Agent Skills

Greenflash analyzes real user-agent conversations to surface where users get blocked, which flows fail, and which interactions drive upgrades or churn. These skills bring that intelligence directly into your coding agent so you can ask questions, get answers, and apply prompt and tool fixes directly when Greenflash recommends them.

## Installation

### Any Agent (Skills CLI)

```bash
npx skills add greenflash-ai/agent-skills
```

### Claude Code (Marketplace)

Run these as **slash commands inside Claude Code** (not in your shell). Start `claude`, then:

```
/plugin marketplace add greenflash-ai/agent-skills
/plugin install greenflash@greenflash-plugins
```

A few things to know:

- These are slash commands inside the Claude Code REPL
- You'll see a "trust this repository" prompt the first time — accept it.
- Plugins activate on the next session, or run `/reload-plugins` to pick them up immediately.

No environment setup required. On first run, the skill activates a fresh API key for you via OAuth — it opens a browser tab, you click **Authorize**, and the key is saved to a gitignored `.greenflash` file in the project. If you can't open a browser (headless, remote SSH), you'll get a paste-key fallback automatically.

## Skills

| Skill                 | Description                                                                                                                           |
| --------------------- | ------------------------------------------------------------------------------------------------------------------------------------- |
| `greenflash`          | Entry point. Auto-detects if Greenflash is set up, offers onboarding if not, and routes your question to the right workflow           |
| `greenflash-health`   | Surface quality trends, anomalies, safety issues, and sentiment across your products                                                  |
| `greenflash-inbox`    | Triage flagged conversations, prioritized by severity                                                                                 |
| `greenflash-users`    | Understand user behavior: who's struggling, who's churning, and what segments look like                                               |
| `greenflash-prompts`  | Find prompt and model quality issues, get optimization recommendations, and apply fixes                                               |
| `greenflash-diagnose` | Root cause analysis for failing tools, user friction, and guardrail violations. Implements fixes directly                             |
| `greenflash-tickets`  | Draft and file tickets in Linear (or your connected ticket provider) from conversation evidence, with a two-step draft → confirm flow |
| `greenflash-update-context`  | Correct a Greenflash analysis you disagree with. Sends one specific correction, in the user's own words, to the product's context notes |

### SDK Integration

| Skill                        | Description                                                                                                          |
| ---------------------------- | -------------------------------------------------------------------------------------------------------------------- |
| `greenflash-onboard-unified` | One-command setup: auto-detects your codebase and walks through all integration steps                                |
| `greenflash-verify`          | Verify your integration is working: API key, SDK, client setup, and data flow                                        |
| `greenflash-onboard`         | Low-level: SDK install + conversation logging only. Most users should reach for `greenflash-onboard-unified` instead   |
| `greenflash-onboard-prompts` | Log system prompts for automatic versioning and optimization                                                         |
| `greenflash-onboard-agentic` | Log structured message types for agent tool calls, reasoning traces, and workflows                                   |
| `greenflash-onboard-events`  | Track business events and link AI conversations to real outcomes like conversions and churn                          |

## Local Development

If you're running the Greenflash API locally, set `GREENFLASH_API_URL` to override the default production URL:

```bash
export GREENFLASH_API_URL=http://localhost:3000/api/v1
```

## Documentation

Full API and plugin documentation: https://docs.greenflash.ai/features/claude-code-skill
