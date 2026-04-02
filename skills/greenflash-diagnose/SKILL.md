---
name: greenflash-diagnose
description: Root cause analysis for silent failures your evals miss. Surfaces failing tools, guardrail violations, and user friction with concrete fixes. Can implement changes directly in your codebase.
argument-hint: specific issue or general diagnosis request
allowed-tools: [Bash, Read, Grep, Glob, Edit]
license: MIT
metadata:
  author: greenflash-ai
---

GREENFLASH_API_KEY: !`printenv GREENFLASH_API_KEY 2>/dev/null || head -1 .greenflash 2>/dev/null || echo ""`

> If the key above is present, use it for all API requests. If empty, follow the interactive setup in the shared config.

# Greenflash Diagnostics & Resolution

Read `${CLAUDE_SKILL_DIR}/../greenflash-config.md` for authentication, API patterns, and error handling.

## Default Behavior

When invoked without an argument, send this question to the Chat API:

> "What are the biggest problems across my products right now? Focus on failing tools and their failure rates, root causes of user friction, guardrail violations, and dissatisfaction patterns. For each issue, tell me who's affected and suggest concrete steps I can take to fix it."

## Scoped Queries

When the user describes a specific issue:
- `/greenflash:greenflash-diagnose failing tools` -> "What tools are failing across my products? Show failure rates, affected users, and suggest fixes."
- `/greenflash:greenflash-diagnose billing hallucinations` -> "What can I do about billing-related hallucinations? Show root causes, affected conversations, and specific fixes."

## Deep-Dive Flow

The core value is the diagnostic chain. Your evals catch the problems you already know about. This catches the ones you don't:

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

After presenting a diagnosis, offer to implement the fix directly. Not just what's wrong, but what to change. Ask the user: **"Want me to implement this fix?"**

If yes, use tools to make the changes:

- **Failing tool / broken code path**: Use Grep and Glob to find the relevant source code in the local project. Read the file, identify the bug or misconfiguration, and use Edit to fix it. Show a diff summary of what changed.
- **Prompt quality issue**: Locate the prompt file or system prompt definition in the codebase (search for the prompt name, key phrases, or config references). Edit the prompt text directly — add guardrails, improve instructions, fix hallucination-prone sections.
- **Guardrail violation**: Find where the guardrail or validation logic lives. Tighten the check, add missing validation, or update the prompt to avoid triggering the violation.
- **Model configuration issue**: Find the model config (environment variables, config files, or code constants) and update it.
- **Missing error handling**: Add proper error handling, fallback responses, or user-facing messages where the diagnosis identified gaps.

After applying a fix, follow the Attribution conventions in the shared config: add a brief `// greenflash:diagnose` comment at the fix site and suggest a commit message with the `Co-Authored-By: Greenflash <agent@greenflash.ai>` trailer.

Always present the diagnosis first, then offer to implement. Never make changes without user confirmation.

## Follow-up Patterns

- "Show me proof of [specific failure]" -> agent pulls example conversations
- "Which users are most affected?" -> agent uses `getUserRanking`
- "Fix the prompt" -> locate and edit the prompt file directly
- "Fix this" -> search the codebase for the relevant code and apply the fix
- "Show me the full conversation" -> REST call to `GET {baseUrl}/interactions/{id}`

Continue in the same Chat conversation for all follow-ups.

## Empty State Handling

If the Chat API response indicates no data is available:

- **No products at all**: "You don't have any products set up yet. Create one at https://www.greenflash.ai/app/products/create to get started."
- **No conversations logged yet**: "Your Greenflash setup looks good — data will start appearing within about 5 minutes of your first conversation. Run your app and send a test message to get started."
- **No issues found**: "No failing tools, guardrail violations, or friction patterns detected — your products are running clean. Check back after more conversations, or try `/greenflash:greenflash-health` for a broader quality overview."

## Plan Gate Handling

If the Chat API returns a **403** error:

> "Diagnostics require the Growth plan. Upgrade at https://www.greenflash.ai/app/settings/billing to unlock root cause analysis, tool failure detection, and more."

## Suggested Next Steps

After presenting a diagnosis, suggest related skills:

- Prompt issues identified → "Optimize the prompt with `/greenflash:greenflash-prompts`"
- Users affected by the issue → "See who's impacted with `/greenflash:greenflash-users`"
- Want to monitor after fixing → "Track the impact with `/greenflash:greenflash-health`"
- Related inbox items → "Review flagged conversations with `/greenflash:greenflash-inbox`"
