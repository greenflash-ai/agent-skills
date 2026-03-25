---
name: greenflash-prompts
description: Evaluate prompt and model performance — quality issues, optimization recommendations, model comparison
argument-hint: prompt name/ID, model name, or general question
license: MIT
metadata:
  author: greenflash-ai
  version: "1.0.0"
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

## Implementation

After presenting optimization recommendations, offer to implement them directly. Ask the user: **"Want me to apply these changes?"**

If yes, use Claude Code's tools to make the edits:

- **Prompt quality fix**: Use Grep/Glob to find the prompt file or system prompt definition in the codebase (search for the prompt name, key phrases, or template variables). Edit the prompt text directly — improve instructions, add examples, tighten constraints, remove hallucination-prone phrasing.
- **Model switch**: Find where the model is configured (env vars, config files, API call parameters) and update the model identifier. Note any cost/latency tradeoffs when making the change.
- **Hallucination mitigation**: Locate the relevant prompt and add grounding instructions — cite-source requirements, factual constraints, or explicit "if unsure, say so" directives.
- **Missing guardrails**: Add output validation, content filtering instructions, or structured output constraints to the prompt.

Always present the analysis first, then offer to implement. Never make changes without user confirmation.

## Follow-up Patterns

- "Which model should I use for X?" -> agent uses `getModelMetrics` + recommendations
- "What prompt changes would improve quality?" -> agent pulls from prompt analysis, then offers to apply them
- "Compare gpt-4o vs claude-3.5" -> agent uses `compareProducts` or model comparison tools
- "Fix this prompt" -> locate the prompt file and edit it directly
- "Switch to [model]" -> find and update the model configuration

Continue in the same Chat conversation for all follow-ups.
