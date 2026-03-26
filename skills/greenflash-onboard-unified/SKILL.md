---
name: greenflash-onboard-unified
description: One-command Greenflash setup. Auto-detects your codebase, walks through SDK installation, conversation logging, system prompt tracking, agentic message types, and business event tracking — all in one guided flow. Use instead of running onboarding skills individually. Triggers on "set up Greenflash", "get started with Greenflash", "add Greenflash", "onboard", or "integrate Greenflash".
argument-hint: optional language hint (python or typescript)
license: MIT
metadata:
  author: greenflash-ai
---

# Greenflash Unified Onboarding

Read `../greenflash-config.md` (relative to this skill's directory) for authentication, API patterns, and error handling.

## Purpose

This skill walks the user through a complete Greenflash setup in one guided flow. It detects the codebase structure, identifies which features are relevant, and delegates to the appropriate sub-skills — so the user doesn't need to know about or invoke 4 separate onboarding skills.

## Flow Overview

1. Account & API key check
2. Product check
3. Codebase analysis
4. Present setup plan
5. Execute approved steps (delegate to sub-skills)
6. Verify everything works
7. Completion summary

---

## Step 1: Account & API Key Check

Resolve the API key using the authentication flow in the shared config.

If **no key is found** (no env var, no `.greenflash` file), gently suggest creating an account:

> "It looks like you don't have a Greenflash API key yet. You can create a free account at https://www.greenflash.ai/sign-up — it takes about 30 seconds. Once you're signed up, grab your API key from https://www.greenflash.ai/app/settings/developers?section=api-keys and I'll get you set up."

Wait for the user to provide a key. Once provided, follow the standard config flow (write to `.greenflash`, gitignore guard).

**Validate the key:** Call `GET {baseUrl}/products?limit=1`.
- **200**: Key is valid — proceed
- **401**: Key is invalid — tell the user and ask them to check it
- **403**: Key is valid but on a limited plan — proceed (SDK logging works on all plans)
- **Network error**: Cannot reach the API — ask the user to check their connection

If the key is invalid, do not proceed. Wait for a valid key.

## Step 2: Product Check

Call `GET {baseUrl}/products` to list the user's products.

If **no products exist** (empty array):

> "You'll need a Greenflash product to start logging conversations. Create one at https://www.greenflash.ai/app/products/create — just give it a name (e.g., 'Customer Support Bot' or 'Sales Assistant') and you'll get a product ID."

Wait for the user to confirm they've created a product, then re-check `GET {baseUrl}/products`.

If **one product exists**, use that product ID automatically.

If **multiple products exist**, ask the user which product this codebase corresponds to. List the product names and let them choose.

Store the selected `product_id` — the sub-skills will need it.

## Step 3: Codebase Analysis

Run these detection scans to build a profile of the target project:

### 3a. Language Detection
Same as the existing `greenflash-onboard` skill:
- **Python**: `pyproject.toml`, `requirements.txt`, `setup.py`, `*.py` files, FastAPI/Django/Flask
- **TypeScript/JavaScript**: `package.json`, `tsconfig.json`, `*.ts`/`*.tsx` files, Next.js/Express

If both detected, ask the user. If an argument was provided, use that.

### 3b. Existing Integration Check
Search for Greenflash SDK imports:
- **Python**: `from greenflash import` or `import greenflash` in any `.py` file
- **TypeScript**: `from 'greenflash'` or `require('greenflash')` in any `.ts`/`.js` file
- Also check dependency files: `package.json`, `pyproject.toml`, `requirements.txt`

If found: skip base SDK setup (step 5a) and note which features are already present.

### 3c. System Prompt Detection
Search for patterns that indicate system prompts are used:
- Variables named `system_prompt`, `systemPrompt`, `system_message`, `SYSTEM_PROMPT`
- OpenAI-style `{"role": "system", "content": ...}` patterns
- Prompt template files (`.txt`, `.md`, `.jinja2` in a `prompts/` or `templates/` directory)
- Anthropic-style system parameter usage

### 3d. Agentic Pattern Detection
Search for patterns that indicate tool-calling or agentic workflows:
- `tool_call`, `function_calling`, `tools=`, `functions=`
- LangChain imports (`from langchain`), LangGraph, CrewAI, AutoGen
- `@tool` decorators
- Multi-step agent frameworks
- Tool definition arrays or function schemas

### 3e. Business Event Detection
Search for patterns that indicate trackable business events:
- Stripe/payment imports (`stripe`, `@stripe/stripe-js`)
- Webhook handlers or event-driven patterns
- Subscription logic (`upgrade`, `downgrade`, `cancel`)
- User signup or conversion flows
- Analytics/tracking calls to other services (Mixpanel, Segment, PostHog)

## Step 4: Present Setup Plan

Based on the analysis, present a numbered plan with clear priority levels so the user knows what matters most:

```
Based on your codebase, here's what I can set up:

1. [Required]     Conversations — install the Greenflash SDK, create a client, and wire up conversation logging. This is the foundation — everything else builds on it.
2. [Recommended]  System prompts — I found system prompts in src/prompts/agent.ts. Logging these unlocks prompt versioning and optimization insights. Strongly recommended.
3. [Recommended]  Agentic messages — I see tool-calling patterns in src/agent/executor.ts. Structured message types give visibility into tool calls and reasoning traces.
4. [Optional]     Business events — No obvious event hooks found, but you can link conversations to outcomes like upgrades or churn. Can be added later with /greenflash:greenflash-onboard-events.

Which of these would you like to include?
```

**Priority levels:**
- **Required**: Conversation logging is the foundation — without it, nothing else works. Always include unless already integrated.
- **Recommended**: System prompt tracking and agentic messages (when detected) are strongly recommended. They significantly improve the quality of Greenflash's analysis. Mark as recommended when patterns are found in the codebase, optional when they're not.
- **Optional**: Business events and any feature where no patterns were detected. Useful but not needed to get value from Greenflash.

**Rules for the plan:**
- Steps where patterns were detected: include briefly where the patterns were found, mark as **Recommended**
- Steps where no patterns detected: mark as **Optional** with a note that it can be added later
- Conversation logging is always step 1 and always **Required** unless already integrated
- If base SDK is already integrated, start from step 2 and note that conversations are already set up
- Explicitly ask the user which items they want to include — don't assume

Wait for user confirmation before proceeding.

## Step 5: Execute Approved Steps

For each approved step, delegate to the corresponding sub-skill. Run them **sequentially** — each step builds on the previous one.

### 5a. Base SDK Integration
Invoke `/greenflash:greenflash-onboard` with the detected language.

After completion, confirm: "SDK is set up and conversation logging is wired in. Moving on to the next step..."

### 5b. System Prompt Tracking
Invoke `/greenflash:greenflash-onboard-prompts` with the detected language.

After completion, confirm: "System prompt tracking is configured. Moving on..."

### 5c. Agentic Messages
Invoke `/greenflash:greenflash-onboard-agentic` with the detected language.

After completion, confirm: "Structured message types are set up for your agent workflows. Moving on..."

### 5d. Business Events
Invoke `/greenflash:greenflash-onboard-events` with the detected language.

After completion, confirm: "Business event tracking is configured."

## Step 6: Verification

After all approved steps are complete, invoke `/greenflash:greenflash-verify` to confirm everything is set up correctly.

Present the verification results to the user.

## Step 7: Completion Summary

Present a final summary:

```
Greenflash Setup Complete
==========================
What was set up:
- [list of completed steps with file paths]

Files modified:
- [list of files created or changed]

Environment:
- GREENFLASH_API_KEY — set in .greenflash (add to .env for production)
- Product ID: [product_id used]

What's next:
- Run your app and send a test message — you'll see your first insights in the Greenflash dashboard within about 5 minutes
- Try /greenflash:greenflash-health to check on your product's health
- Try /greenflash:greenflash-inbox to see flagged conversations
```

Suggest a commit message following the attribution conventions from the shared config:

```
feat: add Greenflash conversation analytics

Integrates the Greenflash SDK for conversation logging[, system prompt tracking][, agentic message types][, business event tracking].

Co-Authored-By: Greenflash <agent@greenflash.ai>
```
