---
name: greenflash-onboard-agentic
description: Log structured message types for agentic workflows: tool calls, reasoning traces, and multi-step chains. Gives Greenflash richer visibility into agent behavior for analysis and optimization. Use when the user has Greenflash integrated and wants to log agent tool calls, reasoning steps, chain-of-thought, or structured messages instead of plain text.
argument-hint: optional language hint (python or typescript)
license: MIT
metadata:
  author: greenflash-ai
---

# Greenflash Agentic Messages Integration

Read `../greenflash-config.md` (relative to this skill's directory) for authentication, API patterns, and error handling.

## Purpose

This skill upgrades an **existing** Greenflash SDK integration to support **agentic workflows** with structured message types for tool calls, reasoning steps, observations, and multi-step chains. This gives Greenflash richer visibility into agent behavior so it can surface the silent failures that evals miss: tools that fail quietly, reasoning that loops, and steps that confuse users.

**Prerequisite:** The codebase must already have a working Greenflash integration (client + message logging). If not, run `/greenflash:greenflash-onboard` first.

## Language Detection

Detect the project language:

1. **Python** — look for existing `greenflash` imports, `greenflash_client.py`, or `from greenflash import`
2. **TypeScript** — look for existing `greenflash` imports, `greenflash-client.ts`, or `import Greenflash from 'greenflash'`

If an argument is provided (e.g., `/greenflash:greenflash-onboard-agentic python`), use that.

---

## Core Concepts

### 1. Message Types

Instead of passing everything as plain text, use `message_type` (Python) / `messageType` (TypeScript) to describe what the agent is doing:

| Type | Description |
|------|-------------|
| `user_message` | Input from the end user |
| `assistant_message` | Response from the assistant |
| `system_message` | System-level instructions |
| `thought` | Internal reasoning or planning |
| `tool_call` | Invocation of an external tool (requires `tool_name`/`toolName`) |
| `observation` | Result returned from a tool |
| `final_response` | The final user-facing response |
| `retrieval` | RAG or document retrieval |
| `memory_read` | Reading from agent memory |
| `memory_write` | Writing to agent memory |
| `chain_start` | Start of a multi-step chain |
| `chain_end` | End of a multi-step chain |
| `embedding` | Embedding operation |
| `tool_error` | Error from a tool |
| `callback` | Callback event |
| `llm` | LLM call |
| `task` | Task execution |
| `workflow` | Workflow execution |

### 2. `content` vs `output` (Critical)

**`content`** — use only for **user-facing** input or output. Anything in `content` runs through all Greenflash analyses (sentiment, quality, safety). If it's not meant to be seen by a user, it should not go here.

**`output`** — use for internal agent details you want visible in the Greenflash UI (thoughts, tool results, internal decisions). Valuable for agentic analysis and debugging, but not treated as user-visible language.

> **Rule of thumb:** If it's not directly user-facing, use `output`, not `content`.

### 3. Modeling Agent Execution

| What | Message Type | Field |
|------|--------------|-------|
| User inputs | `user_message` | `content` |
| Assistant replies shown to users | `assistant_message` or `final_response` | `content` |
| Internal reasoning | `thought` | `output` |
| Tool invocations | `tool_call` | `tool_name`/`toolName` (required), `input` |
| Tool results | `observation` | `output` |
| Errors | `tool_error` | `output` |
| Multi-step flows | `chain_start`, `chain_end`, `task`, or `workflow` | as appropriate |

### 4. Message Relationships (Threading)

Link causally related messages:

- `external_message_id` / `externalMessageId` — unique ID on each message
- `parent_external_message_id` / `parentExternalMessageId` — links child messages to their parent

This preserves execution order and dependency chains.

### 5. Automatic De-duplication

Messages with an `external_message_id`/`externalMessageId` are automatically deduplicated within a conversation. Safe to resend the full conversation history — existing messages are skipped, only new ones inserted.

---

## Submission Patterns

### Option A: Batch Submission

Use when all messages are available at the end of execution.

### Option B: Incremental Submission

Use when messages are produced as the agent runs (streaming, async tool calls).

Choose the pattern that fits the existing code — don't force a pattern.

---

## Python Implementation

### Batch Example (Sync)

```python
client.messages.create(
    product_id="YOUR_PRODUCT_ID",
    external_user_id="USER_ID",
    external_conversation_id="CONVO_ID",
    messages=[
        {"role": "user", "content": "I got charged twice for my subscription this month."},
        {
            "external_message_id": "thought-1",
            "message_type": "thought",
            "output": "Customer reports double charge. Need to verify and pull account details.",
        },
        {
            "external_message_id": "tool-1",
            "message_type": "tool_call",
            "tool_name": "account_details",
            "input": {"search_query": "Michael Chen", "include_billing_history": True},
        },
        {
            "external_message_id": "obs-1",
            "message_type": "observation",
            "parent_external_message_id": "tool-1",
            "output": {
                "recent_charges": [
                    {"date": "2024-01-01", "amount": 49.99, "status": "completed"},
                    {"date": "2024-01-01", "amount": 49.99, "flag": "potential_duplicate"},
                ],
            },
        },
        {
            "role": "assistant",
            "content": "I found your account and can see you were charged $49.99 twice on January 1st. Let me process your refund right now.",
        },
    ],
)
```

### Batch Example (Async fire-and-forget)

```python
asyncio.create_task(client.messages.create(
    product_id="YOUR_PRODUCT_ID",
    external_user_id="USER_ID",
    external_conversation_id="CONVO_ID",
    messages=[...],  # Same array as above
))
```

### Incremental Example (Sync)

```python
# 1. User message
client.messages.create(
    product_id="YOUR_PRODUCT_ID",
    external_user_id="USER_ID",
    external_conversation_id="CONVO_ID",
    messages=[{"role": "user", "content": "I got charged twice for my subscription this month."}],
)

# 2. Agent reasoning
client.messages.create(
    product_id="YOUR_PRODUCT_ID",
    external_user_id="USER_ID",
    external_conversation_id="CONVO_ID",
    messages=[{
        "external_message_id": "thought-1",
        "message_type": "thought",
        "output": "Customer reports double charge. Need to pull account details.",
    }],
)

# 3. Tool call
client.messages.create(
    product_id="YOUR_PRODUCT_ID",
    external_user_id="USER_ID",
    external_conversation_id="CONVO_ID",
    messages=[{
        "external_message_id": "tool-1",
        "message_type": "tool_call",
        "tool_name": "account_details",
        "input": {"search_query": "Michael Chen", "include_billing_history": True},
    }],
)

# 4. Tool result
client.messages.create(
    product_id="YOUR_PRODUCT_ID",
    external_user_id="USER_ID",
    external_conversation_id="CONVO_ID",
    messages=[{
        "external_message_id": "obs-1",
        "message_type": "observation",
        "parent_external_message_id": "tool-1",
        "output": {"duplicate_charge_confirmed": True, "refund_eligible": True},
    }],
)

# 5. Final response
client.messages.create(
    product_id="YOUR_PRODUCT_ID",
    external_user_id="USER_ID",
    external_conversation_id="CONVO_ID",
    messages=[{"role": "assistant", "content": "I found the duplicate charge. Processing your refund now."}],
)
```

### Python Notes

- **Sync apps:** Call `client.messages.create(...)` directly
- **Async apps:** Wrap in `asyncio.create_task()` for fire-and-forget
- Use the Python SDK — do **not** make raw HTTP POST requests
- In Python, calling an async function without `await` returns a coroutine that never executes — always use `asyncio.create_task()`

---

## TypeScript Implementation

### Batch Example (Fire-and-forget)

```typescript
client.messages.create({
  productId: "YOUR_PRODUCT_ID",
  externalUserId: "USER_ID",
  externalConversationId: "CONVO_ID",
  messages: [
    { role: "user", content: "I got charged twice for my subscription this month." },
    {
      externalMessageId: "thought-1",
      messageType: "thought",
      output: "Customer reports double charge. Need to verify and pull account details.",
    },
    {
      externalMessageId: "tool-1",
      messageType: "tool_call",
      toolName: "account_details",
      input: { searchQuery: "Michael Chen", includeBillingHistory: true },
    },
    {
      externalMessageId: "obs-1",
      messageType: "observation",
      parentExternalMessageId: "tool-1",
      output: {
        recentCharges: [
          { date: "2024-01-01", amount: 49.99, status: "completed" },
          { date: "2024-01-01", amount: 49.99, flag: "potential_duplicate" },
        ],
      },
    },
    {
      role: "assistant",
      content: "I found your account and can see you were charged $49.99 twice on January 1st. Let me process your refund right now.",
    },
  ],
}).catch(err => console.error('Greenflash error:', err));
```

### Incremental Example (Fire-and-forget)

```typescript
// 1. User message
client.messages.create({
  productId: "YOUR_PRODUCT_ID",
  externalUserId: "USER_ID",
  externalConversationId: "CONVO_ID",
  messages: [{ role: "user", content: "I got charged twice for my subscription this month." }],
}).catch(err => console.error('Greenflash error:', err));

// 2. Agent reasoning
client.messages.create({
  productId: "YOUR_PRODUCT_ID",
  externalUserId: "USER_ID",
  externalConversationId: "CONVO_ID",
  messages: [{
    externalMessageId: "thought-1",
    messageType: "thought",
    output: "Customer reports double charge. Need to pull account details.",
  }],
}).catch(err => console.error('Greenflash error:', err));

// 3. Tool call
client.messages.create({
  productId: "YOUR_PRODUCT_ID",
  externalUserId: "USER_ID",
  externalConversationId: "CONVO_ID",
  messages: [{
    externalMessageId: "tool-1",
    messageType: "tool_call",
    toolName: "account_details",
    input: { searchQuery: "Michael Chen", includeBillingHistory: true },
  }],
}).catch(err => console.error('Greenflash error:', err));

// 4. Tool result
client.messages.create({
  productId: "YOUR_PRODUCT_ID",
  externalUserId: "USER_ID",
  externalConversationId: "CONVO_ID",
  messages: [{
    externalMessageId: "obs-1",
    messageType: "observation",
    parentExternalMessageId: "tool-1",
    output: { duplicateChargeConfirmed: true, refundEligible: true },
  }],
}).catch(err => console.error('Greenflash error:', err));

// 5. Final response
client.messages.create({
  productId: "YOUR_PRODUCT_ID",
  externalUserId: "USER_ID",
  externalConversationId: "CONVO_ID",
  messages: [{ role: "assistant", content: "I found the duplicate charge. Processing your refund now." }],
}).catch(err => console.error('Greenflash error:', err));
```

### TypeScript Notes

- Use fire-and-forget (no `await`) with `.catch()` to avoid blocking
- Use the TypeScript SDK — do **not** make raw HTTP POST requests

---

## Backward-Compatible Refactoring

The existing integration may be:
- Logging tool calls, thoughts, and outputs as plain assistant messages
- Stuffing everything into `content`

You should:
- Preserve the original behavior semantically
- Translate into structured messages using the appropriate `message_type`/`messageType`
- Move internal data from `content` to `output`
- Avoid breaking existing flows or assumptions

---

## What to Deliver

1. **Updated implementation** using structured Greenflash messages
2. **Clear separation** between user-visible content (`content`) and internal agent data (`output`, `input`)
3. **Correct usage** of message types for all agent activities
4. **SDK-based** message submission (no raw HTTP)
5. **A submission strategy** (batch vs incremental) that fits the existing code
