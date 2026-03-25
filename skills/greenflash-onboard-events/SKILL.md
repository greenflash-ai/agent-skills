---
name: greenflash-onboard-events
description: Add business event tracking to an existing Greenflash SDK integration — link AI interactions to real business outcomes like conversions, upgrades, and churn. Use when the user has Greenflash integrated and wants to track business events, link conversations to outcomes, measure conversion impact, or attribute revenue to AI interactions.
argument-hint: optional language hint (python or typescript)
license: MIT
metadata:
  author: greenflash-ai
  version: "1.0.0"
---

# Greenflash Business Event Tracking

Read `../greenflash-config.md` (relative to this skill's directory) for authentication, API patterns, and error handling.

## Purpose

This skill extends an **existing** Greenflash SDK integration to track **business events** — connecting AI interactions to real business outcomes like conversions, upgrades, and churn. Events close the loop between conversation quality and business impact.

**Prerequisite:** The codebase must already have a working Greenflash integration (client + message logging). If not, run `/greenflash:greenflash-onboard` first.

## Language Detection

Detect the project language:

1. **Python** — look for existing `greenflash` imports, `greenflash_client.py`, or `from greenflash import`
2. **TypeScript** — look for existing `greenflash` imports, `greenflash-client.ts`, or `import Greenflash from 'greenflash'`

If an argument is provided, use that.

---

## Why Track Events?

- **Attribute Success:** Link specific AI responses directly to business outcomes
- **Validate Quality:** Confirm heuristic signals (sentiment) with tangible outcome data
- **Deepen Insights:** Identify conversation patterns that drive real-world results

---

## When to Send Events

Capture moments that matter — user milestones representing value creation or loss.

### Positive Indicators (Success Signals)
- `signup_completed`, `trial_started`, `task_success`
- `upgrade_purchased`, `meeting_booked`, `lead_converted`

### Negative Indicators (Friction/Churn Signals)
- `cancellation_requested`, `error_encountered`
- `workflow_abandoned`, `refund_requested`

### Contextual Indicators (Usage Context)
- `feature_viewed`, `step_completed`, `usage_logged`

---

## Core Event Fields

| Field | Python | TypeScript | Required | Description |
|-------|--------|------------|----------|-------------|
| Event type | `event_type` | `eventType` | Yes | Name of the event (e.g., `upgrade`) |
| Product ID | `product_id` | `productId` | Yes | Your Greenflash Product ID |
| Conversation ID | `conversation_id` | `conversationId` | No | Link to the conversation that influenced this outcome |
| Influence | `influence` | `influence` | No | `positive`, `negative`, or `neutral` (default) |
| Value | `value` | `value` | Yes | The measurable value (e.g., `"149.00"`) |
| Value type | `value_type` | `valueType` | No | Type of value (`currency`, `count`, `numeric`, `text`) |
| Properties | `properties` | `properties` | No | Additional context as key-value pairs |
| Insert ID | `insert_id` | `insertId` | No | Unique ID for deduplication |

---

## Python Implementation

### Basic Event (Sync)

```python
client.events.create(
    event_type="upgrade",
    product_id=product_id,
    conversation_id=conversation_id,
    influence="positive",
    value="149.00",
    value_type="currency",
    properties={
        "plan": "pro",
        "billing_cycle": "annual",
    },
)
```

### Basic Event (Async fire-and-forget)

```python
asyncio.create_task(client.events.create(
    event_type="upgrade",
    product_id=product_id,
    conversation_id=conversation_id,
    influence="positive",
    value="149.00",
    value_type="currency",
    properties={
        "plan": "pro",
        "billing_cycle": "annual",
    },
))
```

### Use Case Examples

**Customer Support — Post-Chat Upgrade:**
```python
client.events.create(
    event_type="upgrade",
    product_id=product_id,
    conversation_id=support_conversation_id,
    influence="positive",
    value="149.00",
    value_type="currency",
    properties={"plan": "pro", "billing_cycle": "annual", "previous_plan": "free"},
)
```

**Customer Support — Cancellation:**
```python
client.events.create(
    event_type="cancellation_requested",
    product_id=product_id,
    conversation_id=support_conversation_id,
    influence="negative",
    value="49.00",
    value_type="currency",
    properties={"reason": "too_expensive", "plan": "starter", "months_subscribed": 3},
)
```

**Sales — Meeting Booked:**
```python
client.events.create(
    event_type="meeting_booked",
    product_id=product_id,
    conversation_id=outreach_conversation_id,
    influence="positive",
    value="50000",
    value_type="currency",
    properties={"pipeline_stage": "qualification", "engagement_touches": 5},
)
```

**Workflow — Task Success:**
```python
client.events.create(
    event_type="task_success",
    product_id=product_id,
    conversation_id=workflow_session_id,
    influence="positive",
    value="1500",
    value_type="numeric",
    properties={"task_type": "data_extraction", "records_processed": 1500, "time_saved_minutes": 45},
)
```

**Workflow — Error:**
```python
client.events.create(
    event_type="error_encountered",
    product_id=product_id,
    conversation_id=workflow_session_id,
    influence="negative",
    value="1",
    value_type="numeric",
    properties={"error_type": "validation_failure", "error_message": "Invalid date format in row 42"},
)
```

### Python — Sampling

```python
# Sample 10% of page view events
client.events.create(
    event_type="feature_viewed",
    product_id=product_id,
    sample_rate=0.1,
    value="dashboard",
    value_type="text",
    properties={"feature": "dashboard"},
)

# Always capture critical events
client.events.create(
    event_type="upgrade",
    product_id=product_id,
    force_sample=True,
    influence="positive",
    value="149.00",
    value_type="currency",
)
```

### Python — Idempotency

```python
import uuid

event_insert_id = str(uuid.uuid4())

client.events.create(
    event_type="upgrade",
    product_id=product_id,
    conversation_id=conversation_id,
    influence="positive",
    value="149.00",
    value_type="currency",
    insert_id=event_insert_id,
)
```

> For critical events, generate `insert_id` before the request and persist it. Retry with the same ID for exactly-once delivery.

---

## TypeScript Implementation

### Basic Event (Fire-and-forget)

```typescript
client.events.create({
  eventType: "upgrade",
  productId: productId,
  conversationId: conversationId,
  influence: "positive",
  value: "149.00",
  valueType: "currency",
  properties: {
    plan: "pro",
    billingCycle: "annual",
  },
}).catch(err => console.error('Greenflash event error:', err));
```

### Basic Event (Awaited)

```typescript
await client.events.create({
  eventType: "upgrade",
  productId: productId,
  conversationId: conversationId,
  influence: "positive",
  value: "149.00",
  valueType: "currency",
  properties: {
    plan: "pro",
    billingCycle: "annual",
  },
});
```

### Use Case Examples

**Customer Support — Post-Chat Upgrade:**
```typescript
client.events.create({
  eventType: "upgrade",
  productId: productId,
  conversationId: supportConversationId,
  influence: "positive",
  value: "149.00",
  valueType: "currency",
  properties: { plan: "pro", billingCycle: "annual", previousPlan: "free" },
}).catch(err => console.error('Greenflash event error:', err));
```

**Customer Support — Cancellation:**
```typescript
client.events.create({
  eventType: "cancellation_requested",
  productId: productId,
  conversationId: supportConversationId,
  influence: "negative",
  value: "49.00",
  valueType: "currency",
  properties: { reason: "too_expensive", plan: "starter", monthsSubscribed: 3 },
}).catch(err => console.error('Greenflash event error:', err));
```

**Sales — Meeting Booked:**
```typescript
client.events.create({
  eventType: "meeting_booked",
  productId: productId,
  conversationId: outreachConversationId,
  influence: "positive",
  value: "50000",
  valueType: "currency",
  properties: { pipelineStage: "qualification", engagementTouches: 5 },
}).catch(err => console.error('Greenflash event error:', err));
```

**Workflow — Task Success:**
```typescript
client.events.create({
  eventType: "task_success",
  productId: productId,
  conversationId: workflowSessionId,
  influence: "positive",
  value: "1500",
  valueType: "numeric",
  properties: { taskType: "data_extraction", recordsProcessed: 1500, timeSavedMinutes: 45 },
}).catch(err => console.error('Greenflash event error:', err));
```

**Workflow — Error:**
```typescript
client.events.create({
  eventType: "error_encountered",
  productId: productId,
  conversationId: workflowSessionId,
  influence: "negative",
  value: "1",
  valueType: "numeric",
  properties: { errorType: "validation_failure", errorMessage: "Invalid date format in row 42" },
}).catch(err => console.error('Greenflash event error:', err));
```

### TypeScript — Sampling

```typescript
// Sample 10% of page view events
client.events.create({
  eventType: "feature_viewed",
  productId: productId,
  sampleRate: 0.1,
  value: "dashboard",
  valueType: "text",
  properties: { feature: "dashboard" },
}).catch(err => console.error('Greenflash event error:', err));

// Always capture critical events
client.events.create({
  eventType: "upgrade",
  productId: productId,
  forceSample: true,
  influence: "positive",
  value: "149.00",
  valueType: "currency",
}).catch(err => console.error('Greenflash event error:', err));
```

### TypeScript — Idempotency

```typescript
import { randomUUID } from 'crypto';

const eventInsertId = randomUUID();

client.events.create({
  eventType: "upgrade",
  productId: productId,
  conversationId: conversationId,
  influence: "positive",
  value: "149.00",
  valueType: "currency",
  insertId: eventInsertId,
}).catch(err => console.error('Greenflash event error:', err));
```

---

## Linking Events to Conversations

The `conversation_id`/`conversationId` field connects business outcomes to AI interactions. Include it when:

- The event happened during an AI session
- The event happened shortly after an AI interaction
- There's a clear causal relationship between conversation and outcome

```python
# Python: Store conversation ID from message logging
conversation_id = response.conversation_id

# Later...
client.events.create(
    event_type="upgrade",
    product_id=product_id,
    conversation_id=conversation_id,
    influence="positive",
    value="149.00",
    value_type="currency",
)
```

```typescript
// TypeScript: Store conversation ID from message logging
const conversationId = response.conversationId;

// Later...
client.events.create({
  eventType: "upgrade",
  productId: productId,
  conversationId: conversationId,
  influence: "positive",
  value: "149.00",
  valueType: "currency",
}).catch(err => console.error('Greenflash event error:', err));
```

---

## Integration Checklist

- [ ] Identified key business milestones in the application
- [ ] Added event tracking at each milestone using `client.events.create(...)`
- [ ] Linked events to conversations by passing `conversation_id`/`conversationId` when relevant
- [ ] Set appropriate influence (`positive`, `negative`, `neutral`)
- [ ] Included `value` and `value_type`/`valueType` for revenue-impacting events
- [ ] Added useful `properties` for deeper analysis
- [ ] Used fire-and-forget pattern to avoid blocking
- [ ] No disruption to existing message logging behavior

---

## What to Deliver

1. **Event tracking** at key business milestones
2. **Proper conversation linking** where applicable
3. **Correct influence assignment** (positive/negative/neutral)
4. **Value tracking** for revenue-related events
5. **No disruption** to existing message logging behavior

This enables Greenflash to connect AI interactions to real business outcomes, validating conversation quality with tangible results.
