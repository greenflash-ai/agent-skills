# Greenflash Shared Config

## Authentication

Resolve the API key using this priority order:

1. **Environment variable**: Check `$GREENFLASH_API_KEY`
2. **Project config file**: Read the first line of `.greenflash` in the project root (via `cat .greenflash 2>/dev/null`)
3. **Interactive setup**: If neither exists, prompt the user:
   - Tell them: "I need your Greenflash API key to continue. You can find it at https://app.greenflash.ai/settings/api-keys"
   - Wait for the user to provide the key
   - Once provided, write it to `.greenflash` in the project root
   - If `.greenflash` is not in `.gitignore`, add it (append `\n.greenflash` to `.gitignore`)
   - Confirm: "API key saved to .greenflash — you won't need to enter it again for this project."

All requests use `Authorization: Bearer {key}` header.

## API Base URL

`https://app.greenflash.ai/api/v1`

> **Local development**: Override with `$GREENFLASH_API_URL` if you're running the API locally.

## Chat API Pattern

Primary interface for all analytical questions:

**Request:** `POST {baseUrl}/chat`
- Headers: `Authorization: Bearer {key}`, `Accept: text/event-stream`, `Content-Type: application/json`
- Body: `{ "question": "...", "messages": [...], "conversationId": "...", "productId": "...", "context": "..." }`
- `messages` is an array of `{ "role": "user" | "assistant", "content": "..." }` from prior turns
- `conversationId` is omitted on first turn, captured from `done` event

**SSE Events:**
- `tool_call` — `{ step, toolName, displayName }` — show progress: `[step N] displayName...`
- `tool_result` — `{ step, toolName, displayName }` — note tool completion
- `text_delta` — `{ text }` — concatenate into the response string
- `done` — `{ conversationId, status, usage: { toolCalls, tools[] } }` — store `conversationId` for follow-ups
- `error` — `{ error, code }` — surface to user

## Message Window Management

- Maintain `conversationId` and `messages` across invocations within this Claude Code session
- First turn: omit `conversationId`, capture from `done` event
- Follow-ups: include `conversationId` and `messages`
- When `messages` exceeds 6 entries: summarize older messages into a single assistant message prefixed with `[Context summary]:` (keep under 500 characters), retain only the last 6 verbatim
- On topic/workflow switch: clear both `conversationId` and `messages` completely

## REST Pattern

For targeted lookups (get entity by ID, list entities):
- `GET {baseUrl}/{resource}` or `GET {baseUrl}/{resource}/{id}`
- Headers: `Authorization: Bearer {key}`, `Accept: application/json`
- Parse JSON response, check `success` field

## Error Handling

- **401**: "Invalid API key. Check your key at https://app.greenflash.ai/settings/api-keys"
- **403**: "This feature requires a Growth plan or higher. Upgrade at https://greenflash.ai/pricing"
- **429**: "Rate limit reached. Try again in a few minutes."
- **404**: "Not found. Check the ID and try again."
- **SSE `error` event**: Surface the error message directly to the user
- **Network failure**: "Could not reach the Greenflash API. Check your connection and API key."
- **Partial SSE failure** (stream drops before `done`): Show whatever text was received, note it may be incomplete, offer to retry
