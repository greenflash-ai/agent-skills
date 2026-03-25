---
name: greenflash-onboard-prompts
description: Upgrade an existing Greenflash SDK integration to log system prompts for prompt optimization, versioning, and component-level analysis. Use when the user already has Greenflash integrated and wants to start logging system prompts, track prompt versions, add prompt components, or enable prompt optimization features.
argument-hint: optional language hint (python or typescript)
license: MIT
metadata:
  author: greenflash-ai
  version: "1.0.0"
---

# Greenflash System Prompt Logging

Read `../greenflash-config.md` (relative to this skill's directory) for authentication, API patterns, and error handling.

## Purpose

This skill upgrades an **existing** Greenflash SDK integration to log **system prompts** alongside conversation messages. This enables Greenflash to automatically version, analyze, and optimize system prompts based on real conversation data.

**Prerequisite:** The codebase must already have a working Greenflash integration (client + message logging). If not, run `/greenflash:greenflash-onboard` first.

## Language Detection

Detect the project language:

1. **Python** — look for existing `greenflash` imports, `greenflash_client.py`, or `from greenflash import`
2. **TypeScript** — look for existing `greenflash` imports, `greenflash-client.ts`, or `import Greenflash from 'greenflash'`

If an argument is provided, use that.

---

## Two Valid Formats

Choose based on the complexity of the prompt being submitted.

### Option A: Simple String Prompt

Use when the system prompt is a **single static string**.

**Python (sync):**
```python
client.messages.create(
    external_user_id=user_id,
    external_conversation_id=convo_id,
    product_id=product_id,
    system_prompt="You are a helpful assistant. Be concise and friendly.",
    messages=[...],
)
```

**Python (async fire-and-forget):**
```python
asyncio.create_task(client.messages.create(
    external_user_id=user_id,
    external_conversation_id=convo_id,
    product_id=product_id,
    system_prompt="You are a helpful assistant. Be concise and friendly.",
    messages=[...],
))
```

**TypeScript (fire-and-forget):**
```typescript
client.messages.create({
  externalUserId: userId,
  externalConversationId: convoId,
  productId: productId,
  systemPrompt: "You are a helpful assistant. Be concise and friendly.",
  messages: [...],
}).catch(err => console.error('Greenflash error:', err));
```

**TypeScript (awaited):**
```typescript
await client.messages.create({
  externalUserId: userId,
  externalConversationId: convoId,
  productId: productId,
  systemPrompt: "You are a helpful assistant. Be concise and friendly.",
  messages: [...],
});
```

**Pros:** Easy to implement, automatically tracked & versioned, good for most use cases.
**Cons:** Cannot express structured or dynamic prompts, less granular optimization.

### Option B: Structured Prompt with Components

Use when the prompt includes multiple pieces, variables, dynamic slots, templates, or RAG context.

**Python:**
```python
system_prompt = {
    "external_template_id": "my-assistant-prompt",
    "components": [
        {
            "type": "system",
            "name": "base_instructions",
            "content": "You are a customer support assistant for {{company}}."
        },
        {
            "type": "system",
            "name": "tone_guidelines",
            "content": "Always be professional and empathetic."
        },
        {
            "type": "rag",
            "name": "context",
            "content": dynamic_rag_text,
            "isDynamic": True
        }
    ],
    "variables": {
        "customerName": customer_name,
        "company": company_name
    }
}

# Sync
client.messages.create(
    external_user_id=user_id,
    external_conversation_id=convo_id,
    product_id=product_id,
    system_prompt=system_prompt,
    messages=[...],
)

# Async fire-and-forget
asyncio.create_task(client.messages.create(
    external_user_id=user_id,
    external_conversation_id=convo_id,
    product_id=product_id,
    system_prompt=system_prompt,
    messages=[...],
))
```

**TypeScript:**
```typescript
const systemPrompt = {
  externalTemplateId: "my-assistant-prompt",
  components: [
    {
      type: "system",
      name: "base_instructions",
      content: "You are a customer support assistant for {{company}}."
    },
    {
      type: "system",
      name: "tone_guidelines",
      content: "Always be professional and empathetic."
    },
    {
      type: "rag",
      name: "context",
      content: dynamicRagText,
      isDynamic: true
    }
  ],
  variables: {
    customerName: customerName,
    company: companyName
  }
};

// Fire-and-forget
client.messages.create({
  externalUserId: userId,
  externalConversationId: convoId,
  productId: productId,
  systemPrompt: systemPrompt,
  messages: [...],
}).catch(err => console.error('Greenflash error:', err));
```

### Key Notes for Structured Prompts

- **`external_template_id`/`externalTemplateId`** groups all versions under a single lineage. Use a stable, human-readable identifier (e.g., `"customer-support-agent"`). Without this, each unique prompt creates a separate lineage.
- Valid component **`type`** values: `system`, `user`, `tool`, `guardrail`, `rag`, `agent`, `other`
- Each component can have a **`source`**: `customer` (default), `participant`, `greenflash`, or `agent`
- Use `isDynamic: True`/`true` for parts that vary per conversation (RAG context, dynamic instructions)
- Use `variables` to pass template variable values (e.g., `{{companyName}}`)

**Advantages:** Modular prompt structure, component-level optimization, variable interpolation, dynamic content slots for RAG.

---

## Detecting the Right Pattern

Inspect the existing codebase:

- If prompts are **static strings** with no variable logic → use **Simple String Format**
- If prompts are **assembled from pieces**, templates, or dynamic slots → use **Structured Prompt with Components**
- If code emits prompt text **dynamically** as it runs → wrap into structured components with `isDynamic: True`/`true`

---

## Integration Checklist

- [ ] Identified where system prompts are constructed in the codebase
- [ ] Chose the right format (simple string vs structured components)
- [ ] Added `system_prompt`/`systemPrompt` to every `messages.create(...)` call
- [ ] If using structured prompts: set `external_template_id`/`externalTemplateId` for lineage tracking
- [ ] If using structured prompts: marked dynamic components with `isDynamic`
- [ ] If using structured prompts: passed template variables via `variables`
- [ ] Preserved existing message logging behavior
- [ ] Used appropriate fire-and-forget pattern for the language

---

## Tips

- **String prompts** are great to start with — they get metrics and analytics immediately
- **Structured prompts** unlock component-level optimization and versioning
- **Dynamic components** allow RAG and other runtime context to be logged correctly
- **Interpolated variables** let you keep templates generic while capturing personalized conversation data

---

## What to Deliver

- Updated SDK integration that logs system prompts
- Support for both simple string and structured prompt components
- Dynamic data and variables logged properly
- No disruption to existing message logging behavior

This unlocks Greenflash's prompt optimization capabilities across your AI products.
