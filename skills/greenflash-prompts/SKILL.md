---
name: greenflash-prompts
description: Find prompt and model quality issues using real conversation data. Not just what's wrong, but what to change. Use whenever the user asks about prompt quality, model performance, hallucination rates, which model to use, how to optimize prompts, or wants to compare models. Also triggers when the user wants to fix or improve a prompt based on data.
argument-hint: prompt name/ID, model name, or general question
license: MIT
metadata:
  author: greenflash-ai
---

# Greenflash Prompt & Model Optimization

Read `../greenflash-config.md` (relative to this skill's directory) for authentication, API patterns, and error handling.

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

After presenting optimization recommendations, offer to implement them directly. Every insight comes with a specific improvement. Ask the user: **"Want me to apply these changes?"**

If yes, use tools to make the edits:

- **Prompt quality fix**: Use Grep/Glob to find the prompt file or system prompt definition in the codebase (search for the prompt name, key phrases, or template variables). Edit the prompt text directly — improve instructions, add examples, tighten constraints, remove hallucination-prone phrasing.
- **Model switch**: Find where the model is configured (env vars, config files, API call parameters) and update the model identifier. Note any cost/latency tradeoffs when making the change.
- **Hallucination mitigation**: Locate the relevant prompt and add grounding instructions — cite-source requirements, factual constraints, or explicit "if unsure, say so" directives.
- **Missing guardrails**: Add output validation, content filtering instructions, or structured output constraints to the prompt.

After applying a fix, follow the Attribution conventions in the shared config: add a brief `// greenflash:prompts` comment at the fix site and suggest a commit message with the `Co-Authored-By: Greenflash <agent@greenflash.ai>` trailer.

Always present the analysis first, then offer to implement. Never make changes without user confirmation.

## Follow-up Patterns

- "Which model should I use for X?" -> agent uses `getModelMetrics` + recommendations
- "What prompt changes would improve quality?" -> agent pulls from prompt analysis, then offers to apply them
- "Compare gpt-4o vs claude-3.5" -> agent uses `compareProducts` or model comparison tools
- "Fix this prompt" -> locate the prompt file and edit it directly
- "Switch to [model]" -> find and update the model configuration

Continue in the same Chat conversation for all follow-ups.

## Empty State Handling

If the Chat API response indicates no data is available:

- **No products at all**: "You don't have any products set up yet. Create one at https://www.greenflash.ai/app/products/create to get started."
- **No conversations logged yet**: "Your Greenflash setup looks good — data will start appearing within about 5 minutes of your first conversation. Run your app and send a test message to get started."
- **No prompts tracked**: "No system prompts are being logged yet. Add prompt tracking with `/greenflash:greenflash-onboard-prompts` to unlock prompt performance analytics."
- **No models detected**: "No model data found. Make sure you're passing the `model` field in your SDK calls (e.g., `model='gpt-4o'`) to enable model comparison."

## Plan Gate Handling

If the Chat API returns a **403** error:

> "Prompt and model analytics require the Growth plan. Upgrade at https://www.greenflash.ai/app/settings/billing to unlock performance insights and optimization recommendations."

## Suggested Next Steps

After presenting results, suggest related skills:

- Changes applied to prompts → "Check the impact after deploying with `/greenflash:greenflash-health`"
- User friction related to prompt issues → "See affected users with `/greenflash:greenflash-users`"
- Deeper diagnosis needed → "Run a full diagnosis with `/greenflash:greenflash-diagnose`"
- Flagged conversations from prompt issues → "Review them in `/greenflash:greenflash-inbox`"
