---
name: greenflash-onboard
description: Integrate the Greenflash SDK into a codebase — installs the SDK, creates a client, wires up message logging, and identifies users/orgs
argument-hint: optional language hint (python or typescript)
license: MIT
metadata:
  author: greenflash-ai
  version: "1.0.0"
---

# Greenflash SDK Integration

Read `skills/greenflash-config.md` for authentication, API patterns, and error handling.

## Purpose

This skill integrates the **Greenflash SDK** into the user's codebase from scratch. It installs the package, creates a reusable client, wires up message logging into the main chat flow, and optionally adds user/org identification.

## Language Detection

Before starting, detect the project language:

1. **Python** — look for `pyproject.toml`, `requirements.txt`, `setup.py`, `*.py` files, or frameworks like FastAPI/Django/Flask
2. **TypeScript/JavaScript** — look for `package.json`, `tsconfig.json`, `*.ts`/`*.tsx` files, or frameworks like Next.js/Express

If both are present, ask the user which language to target. If an argument is provided (e.g., `/greenflash:greenflash-onboard python`), use that.

## Async vs Sync Detection (Python Only)

For Python projects, determine the async model:

1. Look for FastAPI/Starlette/aiohttp usage, `async def` route handlers, or an active `asyncio` loop
2. Check if `aiohttp` is already a dependency
3. If none found, treat as **sync**

This determines which client class to use and the fire-and-forget pattern.

---

## Python Integration

### Step 0: Install

```bash
# Async app (aiohttp present or acceptable):
pip install --pre "greenflash[aiohttp]"

# Sync app:
pip install --pre greenflash
```

> The `--pre` flag is required — the Python SDK is currently in pre-release.

### Step 1: Create the client

Create `greenflash_client.py` in a shared module location (e.g., `app/services/`):

**Sync:**
```python
import os
from greenflash import Greenflash

client = Greenflash(api_key=os.environ.get("GREENFLASH_API_KEY"))
```

**Async:**
```python
import os
from greenflash import AsyncGreenflash

client = AsyncGreenflash(api_key=os.environ.get("GREENFLASH_API_KEY"))
```

### Step 2: Add `GREENFLASH_API_KEY` to the environment

Add to `.env` or the project's environment config:
```
GREENFLASH_API_KEY=YOUR_GREENFLASH_API_KEY
```

Do **not** hard-code the key. The SDK reads `GREENFLASH_API_KEY` automatically if not passed explicitly.

### Step 3: Get the Product ID

Ask the user for their **Product ID** from the Greenflash dashboard. This is passed directly in each API call (not as an env var), since a single codebase may serve multiple products.

### Step 4: Analyze the codebase for logging pattern

Before wiring up logging, understand the codebase structure:

1. Where does user input get received?
2. Where does the LLM/assistant response get generated?
3. Are they in the same function or separate?

**Pattern A — Send as turns** (user + assistant in same location):
```python
messages = [
    {"role": "user", "content": user_input},
    {"role": "assistant", "content": assistant_response}
]
asyncio.create_task(client.messages.create(
    product_id="YOUR_PRODUCT_ID",
    external_user_id=user_id,
    external_conversation_id=convo_id,
    messages=messages
))
```

**Pattern B — Send individually** (user input and assistant output in separate locations):
```python
# When user message is received
asyncio.create_task(client.messages.create(
    product_id="YOUR_PRODUCT_ID",
    external_user_id=user_id,
    external_conversation_id=convo_id,
    messages=[{"role": "user", "content": user_input}]
))

# Later, when assistant responds
asyncio.create_task(client.messages.create(
    product_id="YOUR_PRODUCT_ID",
    external_user_id=user_id,
    external_conversation_id=convo_id,
    messages=[{"role": "assistant", "content": assistant_response}]
))
```

For **sync** apps, call `client.messages.create(...)` directly without `asyncio.create_task()`.

> Greenflash automatically orders messages by timestamp across calls, so both patterns work correctly.

**Required fields:** `product_id`, `external_user_id`, `external_conversation_id`, `messages`

**Recommended optional fields:**
- `model` — the AI model used (e.g., `"gpt-4o"`, `"claude-sonnet-4-20250514"`). Enables model comparison in Greenflash.
- `external_organization_id` — links conversations to an org for segment analysis.
- `properties` — dict of custom metadata.

### Step 5: (Optional) Identify users

Call once when a user logs in or as soon as a stable ID is available:

```python
# Async
asyncio.create_task(client.users.create(external_user_id=user_id))

# Sync
client.users.create(external_user_id=user_id)
```

Optional fields: `external_organization_id`, `name`, `email`, `phone`, `properties`.

### Step 6: (Optional) Identify organizations

```python
# Async
asyncio.create_task(client.organizations.create(external_organization_id=org_id))

# Sync
client.organizations.create(external_organization_id=org_id)
```

### Error Handling

```python
import greenflash

try:
    client.messages.create(...)
except greenflash.APIConnectionError as e:
    print(f"Connection error: {e}")
except greenflash.RateLimitError as e:
    print(f"Rate limit exceeded: {e}")
except greenflash.APIStatusError as e:
    print(f"API error {e.status_code}: {e.response}")
except greenflash.APIError as e:
    print(f"API error occurred: {e}")
```

### Python Checklist

- [ ] Installed the correct package (`greenflash` or `greenflash[aiohttp]`)
- [ ] Added `GREENFLASH_API_KEY` to runtime env (not hard-coded)
- [ ] Have Product ID ready from Greenflash dashboard
- [ ] Created `greenflash_client.py` with the appropriate client (`Greenflash` or `AsyncGreenflash`)
- [ ] Analyzed codebase to determine logging pattern (Pattern A or B)
- [ ] Logging messages with required fields: `product_id`, `external_user_id`, `external_conversation_id`, `messages`
- [ ] Passing `product_id` directly in each call (not from an env var)
- [ ] In async paths, using `asyncio.create_task()` for fire-and-forget logging
- [ ] (Optional) Added `client.users.create(...)` where a stable user ID exists
- [ ] (Optional) Added `client.organizations.create(...)` where an org ID exists

---

## TypeScript Integration

### Step 1: Install

```bash
npm install greenflash
# or: yarn add greenflash / pnpm add greenflash
```

### Step 2: Create the client

Create `greenflash-client.ts` in a shared location (e.g., `src/lib/` or `app/services/`):

```typescript
import Greenflash from 'greenflash';

export const client = new Greenflash({
  apiKey: process.env.GREENFLASH_API_KEY,
});
```

### Step 3: Add `GREENFLASH_API_KEY` to the environment

Add to `.env` or equivalent:
```
GREENFLASH_API_KEY=YOUR_GREENFLASH_API_KEY
```

Do **not** hard-code the key.

### Step 4: Get the Product ID

Ask the user for their **Product ID** from the Greenflash dashboard.

### Step 5: Analyze the codebase for logging pattern

Same analysis as Python — find where user input and assistant output live.

**Pattern A — Send as turns:**
```typescript
const messages = [
  { role: 'user' as const, content: userInput },
  { role: 'assistant' as const, content: assistantResponse }
];

client.messages.create({
  productId: 'YOUR_PRODUCT_ID',
  externalUserId: userId,
  externalConversationId: convoId,
  messages: messages
}).catch(err => console.error('Greenflash error:', err));
```

**Pattern B — Send individually:**
```typescript
// When user message is received
client.messages.create({
  productId: 'YOUR_PRODUCT_ID',
  externalUserId: userId,
  externalConversationId: convoId,
  messages: [{ role: 'user', content: userInput }]
}).catch(err => console.error('Greenflash error:', err));

// Later, when assistant responds
client.messages.create({
  productId: 'YOUR_PRODUCT_ID',
  externalUserId: userId,
  externalConversationId: convoId,
  messages: [{ role: 'assistant', content: assistantResponse }]
}).catch(err => console.error('Greenflash error:', err));
```

Use fire-and-forget (no `await`) to avoid blocking LLM responses. The `.catch()` prevents unhandled promise rejections.

**Required fields:** `productId`, `externalUserId`, `externalConversationId`, `messages`

**Recommended optional fields:**
- `model` — the AI model used. Enables model comparison in Greenflash.
- `externalOrganizationId` — links conversations to an org for segment analysis.
- `properties` — object of custom metadata.

### Step 6: (Optional) Identify users

```typescript
client.users.create({
  externalUserId: userId,
  externalOrganizationId: orgId  // optional
}).catch(err => console.error('Failed to identify user:', err));
```

### Step 7: (Optional) Identify organizations

```typescript
client.organizations.create({
  externalOrganizationId: orgId
}).catch(err => console.error('Failed to identify org:', err));
```

### Error Handling

```typescript
import Greenflash from 'greenflash';

try {
  await client.messages.create({...});
} catch (err) {
  if (err instanceof Greenflash.APIConnectionError) {
    console.error('Connection error:', err);
  } else if (err instanceof Greenflash.RateLimitError) {
    console.error('Rate limit exceeded:', err);
  } else if (err instanceof Greenflash.APIError) {
    console.error(`API error ${err.status}:`, err.name);
  } else {
    throw err;
  }
}
```

### TypeScript Checklist

- [ ] Installed package: `npm install greenflash`
- [ ] Added `GREENFLASH_API_KEY` to runtime env (not hard-coded)
- [ ] Have Product ID ready from Greenflash dashboard
- [ ] Created `greenflash-client.ts` with the client
- [ ] Analyzed codebase to determine logging pattern (Pattern A or B)
- [ ] Logging messages with required fields: `productId`, `externalUserId`, `externalConversationId`, `messages`
- [ ] Passing `productId` directly in each call (not from an env var)
- [ ] Using fire-and-forget (no `await`) with `.catch()` to avoid blocking
- [ ] (Optional) Added `client.users.create(...)` where a stable user ID exists
- [ ] (Optional) Added `client.organizations.create(...)` where an org ID exists

---

## Deliverable

A single PR containing:
- The client module (`greenflash_client.py` or `greenflash-client.ts`)
- Minimal logging wired into the main chat flow (using the appropriate pattern)
- Optional user/org identification hooks
- README notes describing env vars and where the logging occurs

## Follow-Up Skills

After the basic integration is working, the user can enhance it with:
- `/greenflash:greenflash-onboard-agentic` — add structured message types for agent workflows
- `/greenflash:greenflash-onboard-prompts` — log system prompts for prompt optimization
- `/greenflash:greenflash-onboard-events` — track business events linked to conversations
