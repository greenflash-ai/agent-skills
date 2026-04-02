---
name: greenflash-inbox
description: Triage flagged conversations prioritized by severity. Surfaces the interactions that need human attention so you stop flying blind. Use whenever the user asks what needs review, what's flagged, what needs attention, wants to see guardrail violations, or asks about conversations flagged for review.
argument-hint: optional filter (e.g. "guardrail violations", "high severity")
license: MIT
metadata:
  author: greenflash-ai
---

# Greenflash Inbox Triage

Read `../greenflash-config.md` (relative to this skill's directory) for authentication, API patterns, and error handling.

## Default Behavior

When invoked without an argument, send this question to the Chat API:

> "What needs my attention in the review inbox? Prioritize by severity and group by trigger type. For the top items, include a brief summary of what happened."

When invoked with a filter (e.g., `/greenflash:greenflash-inbox guardrail violations`):
- Include the filter naturally in the question: "Show me guardrail violations from my review inbox, prioritized by severity."

## Interaction Flow

1. Check authentication per shared config
2. Send the inbox question to `POST {baseUrl}/chat`
3. Stream SSE events with progress indicators
4. Present the response as a prioritized list

## Drill-Down

When the user asks to see more about a specific item (e.g., "tell me more about #3"):
- Continue in the same Chat conversation (pass `conversationId` and `messages`)
- The agent will call `getInboxDetail` or `getConversationDetail` to pull the full analysis

## Transcript Access

When the user asks to see the full transcript of a conversation:
- Use REST directly per the shared config's REST pattern:
  ```bash
  curl -sS --fail-with-body \
    -H "Authorization: Bearer $GREENFLASH_API_KEY" \
    -H "Accept: application/json" \
    "https://www.greenflash.ai/api/v1/interactions/{interactionId}"
  ```
- Parse and display the messages array with role labels

## Action Suggestions

After presenting inbox items, suggest:
- "Want me to diagnose the root cause of [specific issue]?" → `/greenflash:greenflash-diagnose`
- "Want to see the full transcript for any of these?"
- "Want to see which users are most affected?" → `/greenflash:greenflash-users`

## Empty State Handling

If the Chat API response indicates no data is available:

- **No products at all**: "You don't have any products set up yet. Create one at https://www.greenflash.ai/app/products/create to get started."
- **No conversations logged yet**: "Your Greenflash setup looks good — data will start appearing within about 5 minutes of your first conversation. Run your app and send a test message to get started."
- **Inbox is empty**: "No flagged conversations — your product is running clean. Check back later, or try `/greenflash:greenflash-health` for a broader quality overview."

## Plan Gate Handling

If the Chat API returns a **403** error:

> "Inbox analytics require the Growth plan. Upgrade at https://www.greenflash.ai/app/settings/billing to unlock conversation triage and flagging."

## Suggested Next Steps

After presenting inbox items, suggest related skills:

- Issues needing root cause analysis → "Find the root cause with `/greenflash:greenflash-diagnose`"
- User patterns visible → "Check the user's history with `/greenflash:greenflash-users`"
- Prompt-related flags → "Optimize the prompt with `/greenflash:greenflash-prompts`"
- Want broader view → "Check overall product health with `/greenflash:greenflash-health`"
