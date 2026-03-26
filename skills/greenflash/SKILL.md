---
name: greenflash
description: Analyze real user-agent conversations in your Greenflash workspace. Surfaces where users get blocked, which flows fail, and what drives upgrades or churn. Auto-detects if Greenflash is set up in your codebase and offers guided onboarding if not. Use this skill whenever the user asks about their AI product quality, user conversations, what's broken, how prompts are performing, who their users are, or anything related to Greenflash. Also triggers on "how are my products doing", "what needs attention", "check my AI", "any issues", "show me insights", or "set up Greenflash".
argument-hint: your question (e.g. "how are my products doing?")
license: MIT
metadata:
  author: greenflash-ai
---

# Greenflash Router

Read `../greenflash-config.md` (relative to this skill's directory) for authentication, API patterns, and error handling before proceeding.

## Setup

Before doing anything else, resolve the API key using the authentication flow in the shared config. If this is the user's first time, the interactive setup will walk them through it — once the key is saved, all subsequent runs just work.

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
