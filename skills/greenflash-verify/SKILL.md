---
name: greenflash-verify
description: Verify your Greenflash integration is working correctly. Checks API key validity, product setup, SDK installation, client configuration, message logging, and data flow. Use after onboarding or anytime you want to confirm your setup is healthy. Also triggers on "is Greenflash working", "check my setup", "verify integration", or "integration status".
argument-hint: none
license: MIT
metadata:
  author: greenflash-ai
---

# Greenflash Integration Verification

Read `../greenflash-config.md` (relative to this skill's directory) for authentication, API patterns, and error handling.

## Purpose

This skill checks that a Greenflash integration is set up correctly and data is flowing. It runs a series of checks and reports pass/warn/fail status for each, with actionable troubleshooting guidance when something is wrong.

Run this after onboarding, after deploying changes, or anytime you want to confirm your setup is healthy.

## Checks

Run these checks in order. Stop early if a critical check fails (API key, product) since later checks depend on them.

### 1. API Key Validity

Resolve the API key using the authentication flow in the shared config.

- If no key is found (no env var, no `.greenflash` file): **[FAIL]** "No API key found. Set `GREENFLASH_API_KEY` in your environment or run `/greenflash:greenflash-onboard-unified` to get started."
- If a key is found, validate it: `GET {baseUrl}/products?limit=1`
  - **200**: **[PASS]** "API key valid"
  - **401**: **[FAIL]** "API key is invalid. Double-check your key at https://www.greenflash.ai/app/settings/developers?section=api-keys"
  - **Network error**: **[FAIL]** "Could not reach the Greenflash API. Check your connection."

If the key is invalid or missing, stop here — remaining checks require a valid key.

### 2. Product Exists

Call `GET {baseUrl}/products` to list the user's products.

- **Non-empty array**: **[PASS]** "Found N product(s): [list product names]"
- **Empty array**: **[FAIL]** "No products found. Create one at https://www.greenflash.ai/app/products/create — you'll need a product ID to start logging conversations."

If no products exist, stop here — the SDK requires a `product_id`.

### 3. SDK Installed

Detect project language and check for the SDK package:

**Python:**
- Search for `greenflash` in `pyproject.toml`, `requirements.txt`, `requirements-dev.txt`, `setup.py`, `setup.cfg`, or `Pipfile`
- Also check: `pip list 2>/dev/null | grep -i greenflash`
- **Found**: **[PASS]** "SDK installed (greenflash)" — include version if detectable
- **Not found**: **[FAIL]** "Greenflash SDK not installed. Run `pip install --pre greenflash` (or `pip install --pre 'greenflash[aiohttp]'` for async support)"

**TypeScript/JavaScript:**
- Search for `greenflash` in `package.json` dependencies or devDependencies
- Also check: `node_modules/greenflash` exists
- **Found**: **[PASS]** "SDK installed (greenflash)" — include version from package.json if available
- **Not found**: **[FAIL]** "Greenflash SDK not installed. Run `npm install greenflash`"

**Neither detected**: **[WARN]** "Could not detect project language. Check that the SDK is installed manually."

### 4. Client Module Exists

Search the codebase for Greenflash client instantiation:

**Python patterns:**
- `from greenflash import Greenflash` or `from greenflash import AsyncGreenflash`
- `Greenflash(` or `AsyncGreenflash(`

**TypeScript patterns:**
- `import Greenflash from 'greenflash'` or `require('greenflash')`
- `new Greenflash(`

- **Found**: **[PASS]** "Client module found at {file_path}" — show the file where the client is created
- **Not found**: **[FAIL]** "No Greenflash client found in your codebase. Run `/greenflash:greenflash-onboard-unified` to set one up."

### 5. Message Logging Wired

Search for `client.messages.create` (or the equivalent pattern found in step 4) calls in the codebase.

- **Found N calls**: **[PASS]** "Message logging: {N} integration point(s) found in {file_list}"
- **None found**: **[FAIL]** "Message logging isn't wired up. Make sure `client.messages.create()` is called in your chat flow to send conversations to Greenflash."

### 6. Environment Variable Configured

Check if `GREENFLASH_API_KEY` appears in environment/config files:
- `.env`, `.env.local`, `.env.example`, `.env.development`, `.env.production`
- `docker-compose.yml`, `docker-compose.yaml`

- **Found in .env or deployment config**: **[PASS]** "GREENFLASH_API_KEY configured in {file}"
- **Only in `.greenflash` file**: **[WARN]** "API key is only in `.greenflash` (local dev). Add `GREENFLASH_API_KEY` to your `.env` or deployment config for production."
- **Not found anywhere**: **[WARN]** "GREENFLASH_API_KEY not found in any config file. Make sure it's set in your deployment environment."

### 7. Data Flow

Call `GET {baseUrl}/interactions?limit=1&page=1` to check if any conversations have been received.

- **Non-empty array**: **[PASS]** "Data flowing — Greenflash is receiving conversations."
- **Empty array**: **[WARN]** "No conversations received yet. Make sure your app is running, then send a test message. Data appears within about 5 minutes."

## Troubleshooting Guide

If multiple checks fail, provide targeted guidance based on the failure pattern:

| Pattern | Likely Cause | Fix |
|---------|-------------|-----|
| Key valid, SDK not installed | Started account but haven't integrated yet | Run `/greenflash:greenflash-onboard-unified` |
| SDK installed, no client module | Installed package but didn't complete setup | Run `/greenflash:greenflash-onboard-unified` |
| Client exists, no message logging | Client created but not wired into the chat flow | Check where your LLM responses are generated and add `client.messages.create()` calls |
| Everything passes, no data flowing | App not running or hasn't processed a conversation yet | Start the app and send a test message |
| Key valid, wrong product_id | The product_id in code doesn't match any products in the account | List products with `GET /products` and update the product_id in your code |
| Using AsyncGreenflash without await | Sync/async mismatch | If using `AsyncGreenflash`, ensure calls are awaited or use `asyncio.create_task()`. If in a sync context, use `Greenflash` instead. |

## Output Format

Present results as a clear status report:

```
Greenflash Integration Status
==============================
[PASS] API key valid
[PASS] 2 product(s) found: Customer Support Bot, Sales Assistant
[PASS] SDK installed (greenflash@0.4.2)
[PASS] Client module found at src/lib/greenflash_client.py
[PASS] Message logging: 2 integration points in src/chat/handler.py, src/chat/stream.py
[WARN] GREENFLASH_API_KEY only in .greenflash — add to .env for production
[PASS] Data flowing — Greenflash is receiving conversations

Overall: Looking good! Add your API key to .env for production deployments.
```

**Summary line:**
- All pass: "All clear — your Greenflash setup is healthy."
- Warnings only: "Looking good! [brief note about warnings]"
- Any failures: "[N] issue(s) to fix. [Most critical action to take first]"
