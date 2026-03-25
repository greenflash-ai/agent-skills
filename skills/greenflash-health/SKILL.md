---
name: greenflash-health
description: Check your product health — quality trends, anomalies, safety issues, and sentiment. Use whenever the user asks about product status, quality scores, how things are going, what's trending, safety violations, sentiment shifts, or wants a general health check on their AI products.
argument-hint: optional product name or UUID
license: MIT
metadata:
  author: greenflash-ai
  version: "1.0.0"
---

# Greenflash Health & Monitoring

Read `../greenflash-config.md` (relative to this skill's directory) for authentication, API patterns, and error handling.

## Default Behavior

When invoked without an argument, send this question to the Chat API:

> "Give me a health overview of all my products. Highlight anything that needs attention — quality drops, anomalies, safety issues, or negative sentiment trends."

## Scoped Queries

When the user provides a product name (e.g., `/greenflash:greenflash-health Customer Support Bot`):
- Include the product name in the `question` field naturally: "Give me a health overview of Customer Support Bot..."
- Let the Chat agent resolve the name via its `getProductList` tool

When the user provides a UUID:
- Pass it in the `productId` field of the Chat request to scope server-side

## Interaction Flow

1. Check authentication per shared config
2. Send the health question to `POST {baseUrl}/chat`
3. Stream SSE events:
   - Show `[step N] displayName...` for each `tool_call`
   - Concatenate `text_delta` into the response
4. On `done`, store `conversationId` and present the response

## Guided Follow-ups

After presenting the health overview, suggest relevant next steps based on what was surfaced:

- If quality dropped: "Want me to dig into what's driving that?"
- If a product stands out: "Want to focus on [product name]?"
- If safety issues: "Want me to check the guardrail violations?"
- If everything looks healthy: "Want to see your inbox or check specific users?"

These follow-ups continue in the same Chat conversation (pass `conversationId` and `messages`).
