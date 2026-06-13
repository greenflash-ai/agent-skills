---
name: greenflash-update-context
description: Update a Greenflash product's context notes when an analysis is wrong or incomplete. Use when the user says "Greenflash got this wrong", "fix this analysis", "update product context", "add context to my product", "tell Greenflash that X", or wants to correct a sentiment/frustration/quality call. Forwards the user's exact correction to the product context update endpoint, which intelligently merges it into the product's optimization notes.
argument-hint: the correction in the user's own words (e.g. "caps lock is normal communication style for our power users")
allowed-tools:
  - Bash(curl:*)
  - Bash(printenv GREENFLASH_API_KEY)
  - Bash(printenv GREENFLASH_API_URL)
  - Bash(cat .greenflash*)
  - Bash(hostname)
  - Bash(basename:*)
  - Bash(sleep *)
  - Bash(open 'https://www.greenflash.ai/*)
  - Bash(xdg-open 'https://www.greenflash.ai/*)
  - Bash(start 'https://www.greenflash.ai/*)
  - Read(.greenflash)
  - Read(//**/.greenflash)
  - Read(.gitignore)
  - Read(//**/.gitignore)
  - Edit(.greenflash)
  - Edit(//**/.greenflash)
  - Edit(.gitignore)
  - Edit(//**/.gitignore)
  - Write(.greenflash)
  - Write(//**/.greenflash)
  - Write(.gitignore)
  - Write(//**/.gitignore)
  - Read
  - Grep
  - Glob
license: MIT
metadata:
  author: greenflash-ai
---

> Resolve the API key per the shared config's Authentication section before making requests. Do not pre-resolve or paste the literal key into commands — reference `$GREENFLASH_API_KEY` directly in curl when set.

# Greenflash Product Context Update

Read the shared Greenflash config for authentication, API patterns, and error handling. In Claude Code it lives at `${CLAUDE_SKILL_DIR}/../greenflash-config.md`. If that file is not present (e.g. installed via `npx skills add`, which isolates each skill folder), fetch it once from https://raw.githubusercontent.com/greenflash-ai/agent-skills/main/skills/greenflash-config.md and read that instead.

## What This Does

Sends a single, specific correction or addition to a product's context notes via:

`PATCH {baseUrl}/products/{productId}/context`

Use `PATCH` — the operation merges the feedback into the existing context, it does not replace it. Do not use `POST` to this path.

The endpoint runs an LLM merge that updates the prose body of the product's `optimizationNotes` and maintains a capped (last 5) appendix of dated correction lines. The user does **not** see the full updated notes back — only a one-sentence summary of what changed. That's intentional. Don't try to fetch the prose blob to "verify" the merge.

## When to Use

Use this skill when the user is reacting to a specific Greenflash output and wants to correct it:

- "Greenflash got this wrong — caps lock is normal for our power users, not frustration"
- "Add context that our enterprise tier is allowed to ask for refunds without escalation"
- "Tell Greenflash that 'urgent' in our domain means SLA breach, not user emotion"
- "Update the product context: we deliberately use short replies, that's not a quality issue"
- "This frustration analysis is wrong — explain to Greenflash why"

## When NOT to Use — Refuse and Redirect

Do not call this endpoint in any of these cases. Explain why and point to the right place.

1. **The user's complaint is a Greenflash setup problem, not an analysis problem.** Examples:
   - "Greenflash isn't seeing my prompts" → setup issue, run `/greenflash:greenflash-onboard-prompts` or `/greenflash:greenflash-verify`.
   - "It's analyzing the wrong product" → product mapping issue, check the `product_id` in SDK calls via `/greenflash:greenflash-verify`.
   - "Webhooks aren't firing" → integration issue, not a context issue.
   - "Conversations aren't being logged" → SDK wiring, run `/greenflash:greenflash-verify`.

   Reply: "That sounds like a setup issue, not a context one. Updating product context won't fix it — let's run `/greenflash:greenflash-verify` first to find what's actually missing."

2. **The feedback is too vague to act on.** Examples of too-vague: "the bot is bad at billing", "make it better", "improve the analysis". Ask follow-ups until you have a concrete, specific correction with the *what* and the *why*. Do NOT invent specifics on the user's behalf to "make it actionable" — if the user says "the bot is bad at billing", ask *what* about billing is wrong, *which conversations* showed it, and *what should the model know instead*.

3. **The feedback contradicts existing product notes without explanation.** If the user says something that's the opposite of a recent prior correction, ask them to reconcile: "You told me last week that ALL CAPS = frustration. Now you're saying caps is normal. Which conversations or context changed your mind?" Don't silently overwrite.

## Interaction Flow

### 1. Get a specific correction

When invoked without an argument, ask:

> "What should I tell Greenflash about your product? Be specific — what was wrong about its analysis, and what should it know instead? For example: 'Caps lock is normal communication style for our power users, not a sign of frustration.'"

When invoked with an argument, treat it as the correction text. Still validate that it's specific enough — if not, ask follow-ups before sending.

**Pass-through rule:** Send the user's correction in **their own words**. Don't paraphrase, soften, or "professionalize" it. The endpoint preserves the verbatim input in an audit trail and uses it to drive the merge — paraphrasing loses signal. Light cleanup (removing filler like "umm" or "you know") is fine; rewording the substance is not.

**Length guard:** Feedback must be 1–4000 characters. If the user's correction is longer, tell them and ask them to tighten it — don't truncate silently.

**One correction per call.** If the user says "also update X and also Y", make it two separate calls (sequentially). Each correction lives as its own line in the audit appendix; batching loses that.

### 2. Resolve the productId

Reuse if already known:

- If the conversation already has a `productId` in scope (e.g., the user came from `/greenflash:greenflash-diagnose` and we were analyzing a specific product), reuse it. Confirm with the user: "Updating context for product `{name}` ({productId}) — correct?"
- If a prior turn used a `conversationId` whose `productId` is known, reuse that.

Otherwise, pick from the list:

- Call `GET {baseUrl}/products`
- If exactly one product exists, use it (still confirm with the user by name).
- If multiple, list product names and IDs and ask the user which one this correction applies to.
- If zero products: "You don't have any Greenflash products yet. Create one in the Greenflash app at https://www.greenflash.ai/app/products/create before updating context (products are app-created only — the API can't create them)."

### 3. Build the request

```bash
curl -sS --fail-with-body -X PATCH \
  -H "Authorization: Bearer $GREENFLASH_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"feedback":"<user verbatim correction>","source":"<optional source tag>"}' \
  "https://www.greenflash.ai/api/v1/products/{productId}/context"
```

`-X PATCH` is required here — curl can't infer PATCH from `-d` the way it infers POST, so this is the one place in the skill family where an explicit `-X` is correct (the shared config's "no `-X POST` when `-d` is present" rule does not apply to PATCH).

(Use `$GREENFLASH_API_URL` as the base if it's set — see the shared config.)

**`source` field — when to include:**

- Reacting to a specific conversation analysis: `"conversation <conversationId>"`
- Reacting to a specific analysis on a conversation: `"<analysis name> on conversation <conversationId>"` (e.g. `"frustration analysis on conversation abc-123"`)
- Reacting to a diagnosis from `/greenflash:greenflash-diagnose`: `"diagnose: <short topic>"` (e.g. `"diagnose: billing hallucinations"`)
- Reacting to a flagged inbox item: `"inbox item <inboxItemId>"`
- General context addition with no specific trigger: omit `source`.

`source` is capped at 200 chars. Trim if longer.

### 4. Show the result

On 200, the response looks like:

```json
{
  "productId": "uuid",
  "summary": "Added clarification that ALL CAPS messages are normal style for power users, not frustration.",
  "updatedAt": "2026-04-26T12:34:56.000Z"
}
```

Show the user:

> "Updated context for product `{name}`: {summary}
>
> You can refine the prose directly at https://www.greenflash.ai/app/products/{productId}/edit if you want to clean it up further."

(Use `$GREENFLASH_API_URL`'s app domain if running locally — strip `/api/v1` to get the app host.)

Do **not** try to fetch the full notes after the update to "show what changed". The summary is the contract.

## Error Handling

- **400** (validation): the response `message` will explain. Most common: feedback empty or > 4000 chars. Surface the message and ask the user to fix.
- **404** (product not found): "That product ID isn't in your account. List products with `GET /products` and pick the right one."
- **401**: follow the shared config's 401 branching.
- **403**: "Updating product context requires the Growth plan. Upgrade at https://www.greenflash.ai/app/settings/billing."
- **429**: rate-limited. "Wait a moment and try again — context updates are rate-limited per product."
- **5xx / network**: surface the message and offer to retry once.

## Examples

**Good correction (specific, verbatim, scoped):**

User: "I just looked at this frustration analysis on conv-abc-123 — Greenflash flagged user as frustrated because they typed in caps. That's just how Sarah writes everything. Caps isn't anger for our power users."

Agent action:
- `feedback`: "Sarah types in all caps as her normal communication style. ALL CAPS messages from power users are not a sign of frustration in our product."
- `source`: "frustration analysis on conversation conv-abc-123"

**Bad correction (too vague — refuse, ask):**

User: "the bot is bad at billing"

Agent: "What specifically about billing? Is it answering wrong, refusing valid requests, hallucinating prices, or something else? And what should it do instead? An example conversation would help."

**Bad correction (setup issue masquerading as context):**

User: "Greenflash isn't seeing my new system prompt"

Agent: "That's a logging gap, not a context one — updating notes won't help. Run `/greenflash:greenflash-onboard-prompts` to wire prompt logging, then `/greenflash:greenflash-verify` to confirm it's flowing."

## Suggested Next Steps

After a successful update:

- "Want to refine the prose directly?" → link to https://www.greenflash.ai/app/products/{productId}/edit
- "Want to see if other conversations were misanalyzed the same way?" → `/greenflash:greenflash-inbox`
- "Want to file a ticket to track this issue?" → `/greenflash:greenflash-tickets`
- "Want to re-diagnose with the new context applied?" → `/greenflash:greenflash-diagnose`
