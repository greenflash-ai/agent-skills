---
name: greenflash-users
description: Understand user behavior from real conversations. Individual lookups, segment health, and cohort analysis. Use whenever the user asks about a specific user, wants to see segments, asks who's frustrated or churning, wants to create a user segment, compare cohorts, or look up user activity and sentiment.
argument-hint: user email/name, segment name, or general question
license: MIT
metadata:
  author: greenflash-ai
---

# Greenflash User & Segment Intelligence

Read `../greenflash-config.md` (relative to this skill's directory) for authentication, API patterns, and error handling.

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

## Creating Segments

When the user wants to create a segment, send a natural language description to the Chat API. The agent will translate it into filter rules and call `createSegment`.

- `/greenflash:greenflash-users create a segment for frustrated enterprise users` -> "Create a segment for users who have frustration >= 0.5 and the property 'plan' equals 'enterprise'"
- `/greenflash:greenflash-users set up a segment for users with high commercial intent` -> "Create a segment for users with commercial intent >= 0.6"
- `/greenflash:greenflash-users make a segment for users seen in the last 7 days with more than 5 conversations` -> "Create a segment for users last seen within 7d who have at least 5 conversations"

If the user hits their plan's segment limit, the agent returns an error with the limit and current count. Surface it and suggest upgrading.

Segment creation is available on **all plans** — the number of custom segments is limited per plan.

## Interaction Flow

1. Check authentication per shared config
2. Determine query type: segment overview, individual user, specific segment, or segment creation
3. Send the appropriate question to `POST {baseUrl}/chat`
4. Stream SSE events with progress indicators
5. Present the response

## Follow-up Patterns

The Chat agent handles these naturally as follow-ups in the same conversation:
- "Who are my most frustrated users?" -> agent uses `getUserRanking`
- "Compare power users vs churning users" -> agent uses `compareSegments`
- "Why is she frustrated?" -> agent uses `getUserTrajectory` and conversation detail
- "Show me their conversations" -> agent uses `getConversationDetail`
- "Create a segment for these users" -> agent uses `createSegment`

Continue in the same Chat conversation for all follow-ups.
