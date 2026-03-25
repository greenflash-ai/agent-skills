---
name: greenflash-prompts
description: Evaluate prompt and model performance — quality issues, optimization recommendations, model comparison
argument-hint: prompt name/ID, model name, or general question
---

# Greenflash Prompt & Model Optimization

Read `skills/greenflash-config.md` for authentication, API patterns, and error handling.

## Default Behavior

When invoked without an argument, send this question to the Chat API:

> "How are my prompts and models performing? Flag any that have quality issues, high hallucination rates, or optimization opportunities. Include specific recommendations."

## Scoped Queries

When the user names a specific prompt or model:
- `/greenflash:greenflash-prompts support-v1` -> "How is the prompt 'support-v1' performing? Include quality metrics, any issues, and specific optimization recommendations."
- `/greenflash:greenflash-prompts gpt-4o` -> "How is the model 'gpt-4o' performing across my products? Compare it to other models I'm using."

## REST Fallback for Config Data

When the user asks for a prompt's content (not analytics), use REST directly:
- `GET {baseUrl}/prompts/{id}` — returns the prompt configuration and content
- This avoids burning a Chat request for a simple lookup

## Interaction Flow

1. Check authentication per shared config
2. Determine query type: general overview, specific prompt, specific model, or content lookup
3. For analytics: send to Chat API
4. For content lookup: use REST directly
5. Stream/present the response

## Follow-up Patterns

- "Which model should I use for X?" -> agent uses `getModelMetrics` + recommendations
- "What prompt changes would improve quality?" -> agent pulls from prompt analysis
- "Compare gpt-4o vs claude-3.5" -> agent uses `compareProducts` or model comparison tools

Continue in the same Chat conversation for all follow-ups.
