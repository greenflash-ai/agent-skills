---
name: greenflash-tickets
description: Draft and file tickets in Linear (or your connected ticket provider) directly from Greenflash conversation evidence. Two-step draft → confirm flow with editable title, description, and labels.
argument-hint: describe what to ticket (e.g. "file a bug for the billing hallucination issue")
allowed-tools: [Bash, Read]
license: MIT
metadata:
  author: greenflash-ai
---

GREENFLASH_API_KEY: !`printenv GREENFLASH_API_KEY 2>/dev/null || head -1 .greenflash 2>/dev/null || echo ""`

> If the key above is present, use it for all API requests. If empty, follow the interactive setup in the shared config.

# Greenflash Ticket Creation

Read `${CLAUDE_SKILL_DIR}/../greenflash-config.md` for authentication, API patterns, and error handling.

## What This Does

Turns Greenflash conversation evidence into real tickets in your connected ticket provider (Linear today; more coming). The agent uses a **two-step flow**:

1. `draftTicket` — the agent assembles a title, description, target team, and label suggestions from the conversation evidence. You review and edit.
2. `createTicket` — you confirm, and the ticket is filed in the provider with a back-link to the originating Greenflash conversation.

This runs entirely through the same Chat API as the other Greenflash skills — no separate endpoints.

## Prerequisites

A ticket provider connection must be active for the organization. If none is set up, `draftTicket` will not appear in the tool stream. Direct the user to connect a provider at https://www.greenflash.ai/app/settings/developers?section=linear.

## Default Behavior

When invoked without an argument, ask the user what they want to ticket:

> "What should I file a ticket for? Describe the issue — or reference a specific flagged conversation, failing tool, or diagnosis we've already looked at."

When invoked with an argument (e.g. `/greenflash:greenflash-tickets billing hallucination affecting enterprise users`), turn it into a chat question like:

> "Draft a ticket for: billing hallucination affecting enterprise users. Pull evidence from the most relevant conversations and label it as a bug."

## Interaction Flow

1. Check authentication per shared config
2. Send the drafting question to `POST {baseUrl}/chat`
3. Stream SSE events with progress indicators (`[step N] displayName...`)
4. When a `tool_result` event for `draftTicket` arrives, it includes an `output` field — **render the draft for user review** (see *Rendering the Draft* below)
5. Ask the user to confirm, edit, or cancel
6. On confirm, send a follow-up message in the same `conversationId` asking the agent to call `createTicket` with the (possibly edited) payload
7. When the `tool_result` for `createTicket` arrives, show the `providerIdentifier` and `providerTicketUrl`

## Rendering the Draft

The `tool_result` event for `draftTicket` carries this shape (see the Chat API docs for full schema):

```json
{
  "step": 2,
  "toolName": "draftTicket",
  "displayName": "Drafting ticket",
  "output": {
    "draft": {
      "provider": "linear",
      "title": "Billing page 500 for enterprise users",
      "description": "...",
      "target": { "teamId": "t_123", "teamName": "Core", "teamKey": "CORE" },
      "labelIds": ["lbl_bug"],
      "source": { "type": "conversation", "conversationId": "conv-abc-123" }
    },
    "availableLabels": [
      { "id": "lbl_bug", "name": "bug", "color": "#f00" }
    ],
    "dedupWarning": null
  }
}
```

Present it to the user like this:

```
Draft ticket (Linear → Core):

  Title:       Billing page 500 for enterprise users
  Description: <first ~200 chars, with "..." if truncated>
  Labels:      bug
  Source:      conversation conv-abc-123

Ready to file? (yes / edit title / edit description / edit labels / cancel)
```

If `dedupWarning` is non-null, show it prominently — there may already be a similar ticket.

## Confirming with Edits

When the user confirms (optionally with edits), send a follow-up message in the same `conversationId`:

> "Please call `createTicket` with the following final payload: `{...edited draft JSON...}`. Use the same source."

Include the draft's `source` object verbatim — this preserves the link back to the originating Greenflash conversation.

When `createTicket` returns, the `tool_result.output` looks like:

```json
{
  "status": "created",
  "providerIdentifier": "LIN-99",
  "providerTicketUrl": "https://linear.app/acme/issue/LIN-99"
}
```

Show the user:

> "Filed **[LIN-99](https://linear.app/acme/issue/LIN-99)** in Linear."

If `status` is `already_exists`, use wording like "Linked to existing ticket [LIN-99](…)" — Greenflash deduped against an open ticket with the same source.

## Scoped Invocations

Common patterns:

- `/greenflash:greenflash-tickets file a bug for tool X failures` — drafts a bug ticket with evidence from the most recent tool failures
- `/greenflash:greenflash-tickets ticket the top inbox item` — drafts from the highest-severity flagged conversation
- `/greenflash:greenflash-tickets from conv-abc-123` — drafts from a specific conversation ID

## Follow-up Patterns

- "Edit the title to X" → resubmit with updated title in the `createTicket` payload
- "Add a priority label" → include the label ID in `labelIds`
- "Change the team to Platform" → update `target.teamId` and `target.teamName`
- "Cancel" → drop the draft, do not call `createTicket`

Continue in the same Chat conversation — pass `conversationId` and `messages`.

## Empty State Handling

- **No ticket provider connected**: "No ticket provider is set up for this organization. Connect one at https://www.greenflash.ai/app/settings/developers?section=linear — Linear is supported today."
- **Provider needs re-auth**: If the `createTicket` result carries `error: "provider_needs_setup"`, tell the user: "The Linear connection needs to be reauthorized. Visit https://www.greenflash.ai/app/settings/developers?section=linear to reconnect."
- **Nothing to ticket**: "I couldn't find strong evidence for a ticket yet. Try `/greenflash:greenflash-inbox` or `/greenflash:greenflash-diagnose` first to surface a specific issue worth filing."

## Plan Gate Handling

If the Chat API returns a **403** error:

> "Ticket creation requires the Growth plan. Upgrade at https://www.greenflash.ai/app/settings/billing to unlock the Chat API and filing tickets from conversation evidence."

## Suggested Next Steps

After filing a ticket, suggest:

- "Want me to diagnose more of the same class of issue?" → `/greenflash:greenflash-diagnose`
- "Want to see who else is affected?" → `/greenflash:greenflash-users`
- "Want to check if this is trending?" → `/greenflash:greenflash-health`
