---
name: greenflash-inbox
description: Triage your review inbox — flagged conversations prioritized by severity. Use whenever the user asks what needs review, what's flagged, what needs attention in the inbox, wants to see guardrail violations, or asks about conversations that were flagged for review.
argument-hint: optional filter (e.g. "guardrail violations", "high severity")
license: MIT
metadata:
  author: greenflash-ai
  version: "1.0.0"
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
- Use REST directly: `GET {baseUrl}/interactions/{interactionId}`
- Parse and display the messages array with role labels

## Action Suggestions

After presenting inbox items, suggest:
- "Want me to diagnose the root cause of [specific issue]?" (invokes greenflash-diagnose)
- "Want to see the full transcript for any of these?"
- "Want to see which users are most affected?" (invokes greenflash-users)
