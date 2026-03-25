---
name: greenflash-diagnose
description: Diagnose issues — failing tools, root causes, guardrail violations, and actionable fixes
argument-hint: specific issue or general diagnosis request
license: MIT
metadata:
  author: greenflash-ai
  version: "1.0.0"
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

## Implementation

After presenting a diagnosis, don't just suggest — offer to implement the fix. Ask the user: **"Want me to implement this fix?"**

If yes, use Claude Code's tools to make the changes:

- **Failing tool / broken code path**: Use Grep and Glob to find the relevant source code in the local project. Read the file, identify the bug or misconfiguration, and use Edit to fix it. Show a diff summary of what changed.
- **Prompt quality issue**: Locate the prompt file or system prompt definition in the codebase (search for the prompt name, key phrases, or config references). Edit the prompt text directly — add guardrails, improve instructions, fix hallucination-prone sections.
- **Guardrail violation**: Find where the guardrail or validation logic lives. Tighten the check, add missing validation, or update the prompt to avoid triggering the violation.
- **Model configuration issue**: Find the model config (environment variables, config files, or code constants) and update it.
- **Missing error handling**: Add proper error handling, fallback responses, or user-facing messages where the diagnosis identified gaps.

Always present the diagnosis first, then offer to implement. Never make changes without user confirmation.

## Follow-up Patterns

- "Show me proof of [specific failure]" -> agent pulls example conversations
- "Which users are most affected?" -> agent uses `getUserRanking`
- "Fix the prompt" -> locate and edit the prompt file directly
- "Fix this" -> search the codebase for the relevant code and apply the fix
- "Show me the full conversation" -> REST call to `GET {baseUrl}/interactions/{id}`

Continue in the same Chat conversation for all follow-ups.
