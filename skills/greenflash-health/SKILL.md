---
name: greenflash-health
description: Surface quality trends, anomalies, safety issues, and sentiment across your AI products. Use for health checks, status overviews, and monitoring what needs attention.
argument-hint: optional product name or UUID
allowed-tools: [Bash, Read]
license: MIT
metadata:
  author: greenflash-ai
---

GREENFLASH_API_KEY: !`printenv GREENFLASH_API_KEY 2>/dev/null || head -1 .greenflash 2>/dev/null || echo ""`

> If the key above is present, use it for all API requests. If empty, follow the interactive setup in the shared config.

# Greenflash Health & Monitoring

Read `${CLAUDE_SKILL_DIR}/../greenflash-config.md` for authentication, API patterns, and error handling.

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

## Empty State Handling

If the Chat API response indicates no data is available (empty results, no products, or no conversations):

- **No products at all**: "You don't have any products set up yet. Create one at https://www.greenflash.ai/app/products/create to get started."
- **No conversations logged yet**: "Your Greenflash setup looks good — data will start appearing within about 5 minutes of your first conversation. Run your app and send a test message to get started."
- **All products healthy, nothing flagged**: "All clear — no quality issues, anomalies, or safety concerns detected across your products. Check back after more conversations come in, or try `/greenflash:greenflash-inbox` to review flagged conversations."

## Plan Gate Handling

If the Chat API returns a **403** error, the user is likely on the Free plan. Tell them:

> "Health analytics require the Growth plan. Upgrade at https://www.greenflash.ai/app/settings/billing to unlock quality trends, anomaly detection, and more."

## Suggested Next Steps

After presenting results, suggest related skills based on what was found:

- Quality drops detected → "Dig deeper with `/greenflash:greenflash-diagnose` to find root causes"
- Users impacted → "See who's affected with `/greenflash:greenflash-users`"
- Prompt issues surfaced → "Optimize prompts with `/greenflash:greenflash-prompts`"
- Inbox items mentioned → "Review flagged conversations with `/greenflash:greenflash-inbox`"
