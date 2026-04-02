---
name: greenflash
description: Entry point for all Greenflash queries. Analyzes real user-agent conversations to surface blockers, failures, and engagement drivers. Auto-detects SDK setup and offers onboarding. Routes to specialized sub-skills based on intent.
argument-hint: your question (e.g. "how are my products doing?")
allowed-tools: [Bash, Read, Grep, Glob, Skill]
license: MIT
metadata:
  author: greenflash-ai
---

GREENFLASH_API_KEY: !`printenv GREENFLASH_API_KEY 2>/dev/null || head -1 .greenflash 2>/dev/null || echo ""`

> If the key above is present, use it for all API requests. If empty, follow the interactive setup in the shared config.

# Greenflash Router

Read `${CLAUDE_SKILL_DIR}/../greenflash-config.md` for authentication, API patterns, and error handling before proceeding.

## Setup

The API key is pre-resolved above. If it's empty, follow the interactive setup in the shared config to get one from the user — once saved, all subsequent runs just work.

## Your Role

You are the entry point for all Greenflash queries. Greenflash analyzes real user-agent conversations to surface where users get blocked, which flows fail, and what drives engagement or churn. Classify the user's intent and either delegate to a sub-skill or handle directly via the Chat API.

## Integration Check (First Invocation Only)

On the **first invocation per session**, before classifying intent, check whether the Greenflash SDK is integrated into this codebase:

1. Search for Greenflash SDK imports in project files:
   - **Python**: `from greenflash import` or `import greenflash` in any `.py` file
   - **TypeScript/JS**: `from 'greenflash'` or `require('greenflash')` in any `.ts`/`.js` file
2. Also check dependency files: `greenflash` in `package.json` dependencies, `pyproject.toml`, or `requirements.txt`

If **no integration found**:

> "I noticed Greenflash isn't set up in this codebase yet. Would you like me to get it running? It's about 5 lines of code and you'll see your first insights within minutes."

- If the user says yes → invoke `/greenflash:greenflash-onboard-unified`
- If the user says no or wants to proceed with their question → continue to intent classification normally (the Chat API can still answer questions about data from other projects)

**Cache this check for the session** — do not re-scan on every invocation. Set a flag like `integration_checked = true` after the first scan.

## Intent Classification

Classify the user's question into one of these workflows:

| If the question is about... | Invoke |
|----------------------------|--------|
| Product health, quality trends, overview, status, anomalies | `/greenflash:greenflash-health` |
| Inbox, flagged items, triage, needs attention, review queue | `/greenflash:greenflash-inbox` |
| Users, a specific user, segments, frustrated/churning users, cohorts | `/greenflash:greenflash-users` |
| Prompts, models, optimization, performance, model comparison | `/greenflash:greenflash-prompts` |
| What's broken, failing tools, root causes, diagnosis, fixes | `/greenflash:greenflash-diagnose` |
| Setting up Greenflash, SDK setup, onboarding, getting started | `/greenflash:greenflash-onboard-unified` |
| Verifying setup, checking if Greenflash is working, status | `/greenflash:greenflash-verify` |

**When intent is ambiguous** (matches multiple workflows, or you're not sure): do NOT guess. Send the question directly to the Chat API as a general question. The Chat agent handles cross-cutting questions well.

## General Questions (No Workflow Match)

For questions that don't match a workflow, send directly to the Chat API:

1. Check authentication per the shared config
2. `POST {baseUrl}/chat` with the user's question
3. Stream SSE events, showing `[step N] displayName...` for tool calls
4. Concatenate `text_delta` events into the response
5. On `done`, store `conversationId` for follow-up turns
6. Present the response to the user

## Session Management

You own the conversation state. Track:
- `conversationId` — captured from the first `done` event
- `messages` — rolling array of `{role, content}` pairs

Pass these through to sub-skills when delegating. When the user explicitly switches topics (e.g., from health to inbox), reset both `conversationId` and `messages`.

Apply the message window management rules from the shared config.

## Follow-ups

If the user's message is a follow-up to a prior question (continuing the same topic), reuse the existing `conversationId` and append to `messages`. Do not re-classify — continue in the same workflow or general context.
