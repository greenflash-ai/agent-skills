---
name: greenflash-diagnose
description: Diagnose issues — failing tools, root causes, guardrail violations, and actionable fixes
argument-hint: specific issue or general diagnosis request
---

# Greenflash Diagnostics & Resolution

Read `skills/greenflash-config.md` for authentication, API patterns, and error handling.

## Default Behavior

When invoked without an argument, send this question to the Chat API:

> "What are the biggest problems across my products right now? Focus on failing tools and their failure rates, root causes of user friction, guardrail violations, and dissatisfaction patterns. For each issue, tell me who's affected and suggest concrete steps I can take to fix it."

## Scoped Queries

When the user describes a specific issue:
- `/greenflash:greenflash-diagnose failing tools` -> "What tools are failing across my products? Show failure rates, affected users, and suggest fixes."
- `/greenflash:greenflash-diagnose billing hallucinations` -> "What can I do about billing-related hallucinations? Show root causes, affected conversations, and specific fixes."

## Deep-Dive Flow

The core value of this skill is the diagnostic chain:

1. **Surface the problem** — the Chat agent identifies failing tools, root causes, violations
2. **Show who's affected** — users, segments, products impacted
3. **Provide evidence** — example conversation transcripts
4. **Recommend a fix** — prompt changes, model switches, tool fixes

When the user asks for evidence (e.g., "show me an example conversation"):
- The Chat agent may use `getConversationDetail` internally
- For full transcript access, use REST: `GET {baseUrl}/interactions/{interactionId}`
- Display messages with role labels and timestamps

## Interaction Flow

1. Check authentication per shared config
2. Send the diagnostic question to `POST {baseUrl}/chat`
3. Stream SSE events with progress indicators
4. Present issues as a prioritized list with severity, impact, and resolution steps

## Follow-up Patterns

- "Show me proof of [specific failure]" -> agent pulls example conversations
- "Which users are most affected?" -> agent uses `getUserRanking`
- "Fix the prompt" -> suggest specific changes or invoke greenflash-prompts
- "Show me the full conversation" -> REST call to `GET {baseUrl}/interactions/{id}`

Continue in the same Chat conversation for all follow-ups.
