---
name: greenflash-users
description: Understand user behavior — individual lookups, segment health, and cohort analysis
argument-hint: user email/name, segment name, or general question
---

# Greenflash User & Segment Intelligence

Read `skills/greenflash-config.md` for authentication, API patterns, and error handling.

## Default Behavior

When invoked without an argument, send this question to the Chat API:

> "Give me an overview of my user segments. Which segments are healthiest? Which have the most friction or frustration? Highlight any individual users that need attention."

## Individual User Queries

When the user provides an identifier (email, name, or external ID):
- Send to Chat: "Tell me about user [identifier]. What's their sentiment, engagement history, segment memberships, and any notable patterns?"

Examples:
- `/greenflash:greenflash-users alice@example.com` -> "Tell me about user alice@example.com..."
- `/greenflash:greenflash-users demo-user-123` -> "Tell me about user demo-user-123..."

## Segment Queries

When the user names a segment:
- `/greenflash:greenflash-users enterprise segment` -> "Tell me about the enterprise segment. What's the health, size, key metrics, and any trends?"

## Interaction Flow

1. Check authentication per shared config
2. Determine query type: segment overview, individual user, or specific segment
3. Send the appropriate question to `POST {baseUrl}/chat`
4. Stream SSE events with progress indicators
5. Present the response

## Follow-up Patterns

The Chat agent handles these naturally as follow-ups in the same conversation:
- "Who are my most frustrated users?" -> agent uses `getUserRanking`
- "Compare power users vs churning users" -> agent uses `compareSegments`
- "Why is she frustrated?" -> agent uses `getUserTrajectory` and conversation detail
- "Show me their conversations" -> agent uses `getConversationDetail`

Continue in the same Chat conversation for all follow-ups.
