# Greenflash Shared Config

## Authentication

**Run this resolution before the first API request of every session.** Do not call any endpoint until a key is resolved — an empty `Authorization: Bearer ` header will return 401 and waste a round trip.

Resolve the API key using this priority order:

1. **Environment variable**: Run `printenv GREENFLASH_API_KEY` — if it outputs a non-empty value, use that as the key
2. **Project config file**: Read the first line of `.greenflash` in the project root (use the Read tool, or `cat .greenflash` via Bash)
3. **Browser activation (preferred)**: If neither exists (or both are empty), run the **device-code activation flow** below. This mints a fresh API key in the browser and saves it to `.greenflash` — no copy/paste required.
4. **Manual fallback**: Use this only if the device-code endpoint returns 404 (older Greenflash deployment) or the user explicitly prefers it. Direct them to https://www.greenflash.ai/app/settings/developers?section=api-keys, wait for them to paste the key, then write the first line of `.greenflash`.

After any path that produces a key, confirm: "API key saved to .greenflash — you won't need to enter it again for this project."

**Gitignore guard**: Whenever `.greenflash` exists on disk (from step 2, 3, or 4), check that `.gitignore` contains `.greenflash`. If not, append `.greenflash` on its own line. This prevents accidental commits of the API key.

Use the **Read** tool to inspect `.gitignore` and the **Edit** tool to append the entry — do not shell out to `grep`/`cat`/`echo`. The skill pre-approves Read/Edit/Write on `.gitignore` but not arbitrary Bash patterns over it, so the dedicated tools avoid an unnecessary permission prompt.

All requests use `Authorization: Bearer {key}` header.

### Device-code activation flow (step 3)

This is an OAuth 2.0 Device Authorization Grant (RFC 8628), the same pattern used by GitHub CLI, gcloud, and others. You orchestrate it directly via curl — no shell loops, no compound commands.

**Step 3a — Initiate.** Single POST to fetch a `device_code` and short `user_code`. Use plain `curl -sS` (without `--fail-with-body`) because the poll endpoint returns 400 with a meaningful body for pending/denied; you'll parse the body either way.

Always include `client_metadata.client_name` so the activation page can identify your tool to the user ("Authorize Claude Code") and brand the minted API key. Without it the page falls back to generic "Authorize API access" copy. Use the canonical brand name of the surface running the skill — e.g. `"Claude Code"`, `"Vercel CLI"`, `"Cursor"`. Don't shorten or improvise.

Optionally include `hostname` and `project` so the activation page can also show the user **which device and project are asking** ("Device: gabes-macbook · Project: greenflash"). Gather these two facts as their own Bash calls:

- **Hostname** — run `hostname` (single command, returns the machine name)
- **Project** — the basename of the project root (read it from the cwd that's already in your skill context — do not shell out to `pwd`)

Then POST with the body. If for any reason you can't gather hostname/project, omit them — only `client_name` is materially important.

```bash
curl -sS -H "Accept: application/json" -H "Content-Type: application/json" -d '{"client_metadata":{"client_name":"Claude Code","hostname":"<host>","project":"<project>"}}' "https://www.greenflash.ai/api/v1/auth/device"
```

> **Local dev**: replace the host with `$GREENFLASH_API_URL` if it's set.

The successful response shape:
```json
{
  "device_code": "<long opaque>",
  "user_code": "ABCD-2345",
  "verification_uri": "https://www.greenflash.ai/activate",
  "verification_uri_complete": "https://www.greenflash.ai/activate?user_code=ABCD-2345",
  "expires_in": 300,
  "interval": 5
}
```

The `user_code` uses an unambiguous alphabet — no `0`, `1`, `I`, or `O` will ever appear.

**Failure modes for this call:**

- **Response is HTML, an empty body, a 404**, or anything else not matching this shape → the activation endpoint isn't deployed on this server. Abandon the device flow and fall through to step 4 (manual fallback).
- **HTTP 429** with body `{"success": false, "error": "Rate limit exceeded", ...}` → you've hit the per-IP rate limit on the initiate endpoint. The response includes a `Retry-After` header in seconds. Tell the user "Greenflash is rate-limiting activation requests from this network — try again in `<Retry-After>` seconds, or paste a key manually." Offer step 4 as an immediate fallback.

**Step 3b — Hand off to the browser.** Display the `user_code` AND the `verification_uri_complete` URL prominently before opening anything. The user has to **visually match** the code shown on the activation page against what's in the terminal — that's the anti-phishing check, and it only works if the code is easy to see.

Use this exact format, **substituting the actual `user_code` and `verification_uri_complete` values from the step 3a response** — do not literally print the placeholders below:

```
<user_code from response>
```

Open: `<verification_uri_complete from response>`

For example, if the API returns `user_code: "ZPUV-EEF3"` and `verification_uri_complete: "https://www.greenflash.ai/activate?user_code=ZPUV-EEF3"`, your output is the fenced block containing exactly `ZPUV-EEF3` followed by `Open: https://www.greenflash.ai/activate?user_code=ZPUV-EEF3`.

Then try to open the URL automatically. Pick the right command for the OS:

- macOS: `open <verification_uri_complete>`
- Linux: `xdg-open <verification_uri_complete>`
- Windows / WSL: `start <verification_uri_complete>` (or skip; just rely on the printed URL)

Run the open command as a single Bash call. If it fails, that's fine — the printed URL is the source of truth. Then tell the user:

> "Confirm the code above matches what you see in the browser, then click **Authorize**. I'll wait."

**Step 3c — Poll.** Call the token endpoint with the `device_code`. The server enforces a 5-second minimum interval. Run `sleep 5` as its own Bash call between polls — never compound it with curl.

```bash
curl -sS -H "Accept: application/json" -H "Content-Type: application/json" -d '{"device_code":"<paste device_code>"}' "https://www.greenflash.ai/api/v1/auth/device/token"
```

(`-d` implies `POST`, so no `-X POST` flag — the project rule forbids combining them.)

Parse the response body — do not rely on the HTTP status code (it's 400 for all non-success cases):

- **Body contains `api_key`** → done. Take the value, save to `.greenflash`, run the gitignore guard, continue.
- **Body is `{"error":"authorization_pending"}`** → user hasn't approved yet. Run `sleep 5`, then poll again. Cap total wait at ~5 minutes (the server-side expiry).
- **Body is `{"error":"slow_down"}`** → polled too fast. Run `sleep 5`, poll again — do not double the interval.
- **Body is `{"error":"access_denied"}`** → user clicked Cancel. Stop polling. Tell the user the activation was cancelled; offer to retry or fall back to manual key entry.
- **Body is `{"error":"expired_token"}`** → code expired or already exchanged. Stop polling. Offer to restart from step 3a.

**Status updates**: While polling, give the user a brief progress note every 2–3 polls (e.g., "Still waiting for approval…"). Don't narrate every single poll.

**Step 3d — Persist.** Once you have an `api_key` (it starts with `gf_`), save it as the first line of `.greenflash`, run the gitignore guard, and confirm: "Saved your new API key to `.greenflash` — you can review or revoke it at https://www.greenflash.ai/app/settings/developers." Don't predict the key name in the confirmation; it's derived server-side from the signed-in user.

**File handling (rotation case)**: `.greenflash` may already exist if the user is rotating a key. To avoid the "File has not been read yet" error from Write, follow this order:

1. Read `.greenflash` first — if it exists, use **Edit** (replace the old key with the new one).
2. If Read returns "file does not exist", use **Write** to create it.

Don't attempt Write directly when the file might exist; it costs an extra tool call when it errors out.

### Using the key in curl commands

- **If `$GREENFLASH_API_KEY` is set**: Reference the variable directly in curl — `Authorization: Bearer $GREENFLASH_API_KEY`. Do NOT resolve the variable first and paste the literal value.
- **If the key came from `.greenflash` or user input**: You will need to paste the literal key value into the curl command. This is fine, but keep the command clean — longer commands with literal keys are where redirect mistakes (`2>&1`) tend to creep in.
- **NEVER** use compound commands to avoid pasting the key (e.g., `KEY=$(cat .greenflash) && curl ...`). Shell operators like `&&`, `||`, `$(...)`, and backticks break permission matching the same way `2>&1` does.

## Account Creation

If the user has no API key and appears to be new to Greenflash, gently suggest creating an account before asking for the key:

- **Signup URL**: https://www.greenflash.ai/sign-up
- **API key page**: https://www.greenflash.ai/app/settings/developers?section=api-keys
- **Product creation**: https://www.greenflash.ai/app/products/create

This is a suggestion, not a blocker — the user may already have a key from a teammate or another project.

## API Base URL

`https://www.greenflash.ai/api/v1`

> **Local development**: Override with `$GREENFLASH_API_URL` if you're running the API locally.

## Request Execution Rules

**Use the URL exactly as specified above.** Do not guess alternative subdomains (e.g., `app.greenflash.ai`, `api.greenflash.ai`). Do not verify, probe, or test the URL before making the actual request. Just call it.

**Keep ALL Bash commands clean — no shell redirects or pipes:**
- **NEVER** append `2>&1`, `2>/dev/null`, `| head`, `| tail`, or any other shell redirects/pipes to ANY command. These contain `>` or `|` characters that break Claude Code's permission pattern matching and cause "Permission denied" errors. This applies to curl commands, cat commands, and everything else.
- Use `curl -sS --fail-with-body` for all requests — this suppresses progress bars and only shows the response body (or error body on failure)
- For SSE streams, use `curl -sS -N` (unbuffered) — do not add `-v` or `--verbose`
- Never use `-X POST` when `-d` is present (curl infers POST automatically)
- curl commands must end with the URL as the final argument — nothing after it
- Do not announce or narrate the API call (no "Let me verify the endpoint" or "Checking API accessibility"). Just make the request silently and present the result

**If a request fails:** Show the HTTP status code and error message from the response body. Do not retry with verbose flags or alternative URLs — follow the error handling rules below instead.

## Chat API Pattern

Primary interface for all analytical questions:

**Request:** `POST {baseUrl}/chat`
- Headers: `Authorization: Bearer {key}`, `Accept: text/event-stream`, `Content-Type: application/json`
- Body: `{ "question": "...", "messages": [...], "conversationId": "...", "productId": "...", "context": "..." }`
- `messages` is an array of `{ "role": "user" | "assistant", "content": "..." }` from prior turns
- `conversationId` is omitted on first turn, captured from `done` event

**Example:**
```bash
curl -sS -N \
  -H "Authorization: Bearer $GREENFLASH_API_KEY" \
  -H "Accept: text/event-stream" \
  -H "Content-Type: application/json" \
  -d '{"question":"..."}' \
  "https://www.greenflash.ai/api/v1/chat"
```

**SSE Events:**
- `tool_call` — `{ step, toolName, displayName }` — show progress: `[step N] displayName...`
- `tool_result` — `{ step, toolName, displayName }` — note tool completion
- `text_delta` — `{ text }` — concatenate into the response string
- `done` — `{ conversationId, status, usage: { toolCalls, tools[], inputTokens, outputTokens } }` — store `conversationId` for follow-ups. Do not show token usage to the user.
- `error` — `{ error, code }` — surface to user

**Presenting SSE output:** Parse the raw SSE stream yourself — do not show the raw `event:` / `data:` lines to the user. Show only the progress steps and the final concatenated response text. If the stream returns an error event, show the error message, not the raw SSE frame.

## Message Window Management

- Maintain `conversationId` and `messages` across invocations within this session
- First turn: omit `conversationId`, capture from `done` event
- Follow-ups: include `conversationId` and `messages`
- When `messages` exceeds 6 entries: summarize older messages into a single assistant message prefixed with `[Context summary]:` (keep under 500 characters), retain only the last 6 verbatim
- On topic/workflow switch: clear both `conversationId` and `messages` completely

## REST Pattern

For targeted lookups (get entity by ID, list entities):
- `GET {baseUrl}/{resource}` or `GET {baseUrl}/{resource}/{id}`
- Headers: `Authorization: Bearer {key}`, `Accept: application/json`
- Parse JSON response, check `success` field

### Pagination (List Endpoints)

All list endpoints (`/interactions`, `/products`, `/prompts`, `/models`, `/segments`, `/users`, `/inbox`) use **Link header pagination**:

- **Query params:** `page` (default 1), `limit` (default varies by endpoint, max 100)
- **Response body:** A flat JSON array of items (not wrapped in an object)
- **Pagination info:** Returned in the `Link` HTTP header using RFC 8288 format:
  ```
  Link: <https://www.greenflash.ai/api/v1/interactions?page=2&limit=50>; rel="next"
  ```
- **No `next` link** means you're on the last page
- To paginate: parse the `Link` header, extract the `rel="next"` URL, and fetch it

**List example:**
```bash
curl -sS --fail-with-body \
  -H "Authorization: Bearer $GREENFLASH_API_KEY" \
  -H "Accept: application/json" \
  "https://www.greenflash.ai/api/v1/interactions?page=1&limit=50"
# Response: [...items...]
# Link header: <.../interactions?page=2&limit=50>; rel="next"
```

**Get-by-ID example:**
```bash
curl -sS --fail-with-body \
  -H "Authorization: Bearer $GREENFLASH_API_KEY" \
  -H "Accept: application/json" \
  "https://www.greenflash.ai/api/v1/interactions/{id}"
```

For creating resources:
- `POST {baseUrl}/{resource}`
- Headers: `Authorization: Bearer {key}`, `Content-Type: application/json`
- Returns 201 on success

### POST /segments — Create Custom Segment

Available on **all plans** (number limited by plan's `maxCustomSegments` quota).

**Body:**
```json
{
  "name": "High-Intent Enterprise Users",
  "description": "Users with high commercial intent from the enterprise plan",
  "icon": "Users",
  "filters": {
    "rules": [
      { "type": "analysis", "field": "commercialIntent", "operator": "gte", "value": 0.6 },
      { "type": "property", "key": "plan", "operator": "eq", "value": "enterprise" }
    ],
    "productIds": ["optional-uuid"],
    "dateRange": { "preset": "30d" }
  }
}
```

**Rule types:** `analysis` (sentiment/frustration/struggle/commercialIntent/cqi/rating), `analysis_flag` (jailbreakDetected/hallucinationDetected/etc.), `property` (user properties), `conversation_property`, `conversation_count`, `last_seen`

**Response (201):** `{ id, name, type, description, icon, filters, createdAt, updatedAt }`

**403:** Segment limit reached — surface the limit and suggest upgrading.

## Plan Requirements

The following endpoints require a **Growth plan or higher**:
- `POST /chat` — streaming chat (also rate-limited per hour)
- `GET /*/analytics` — all analytics endpoints (product, model, prompt, user, organization, segment)

Free-plan API keys will receive a 403 error. If this happens, tell the user: "This feature requires a Growth plan or higher. Upgrade at https://www.greenflash.ai/app/settings/billing"

List/get endpoints (interactions, products, prompts, models, segments, users, inbox) work on all plans.

### Plan Features Reference

Use this when explaining what's available or when a user hits a feature gate:

| Feature | Free | Growth | Enterprise |
|---------|------|--------|------------|
| SDK logging (messages, prompts, events) | Yes | Yes | Yes |
| List/get endpoints (interactions, products, etc.) | Yes | Yes | Yes |
| Custom segments | Limited | More | Unlimited |
| Chat API (analytical questions) | No | Yes | Yes |
| Analytics endpoints | No | Yes | Yes |

- **Upgrade URL**: https://www.greenflash.ai/app/settings/billing
- When a 403 indicates a plan gate, explain what the feature does and how upgrading unlocks it — don't just say "upgrade"

## Attribution

When the agent implements a code fix, include Greenflash attribution so the change is traceable:

- **Inline comment** (at the point of change): `// greenflash:diagnose — <brief reason>` or `// greenflash:prompts — <brief reason>`. One line, only where the fix was applied. Keep it short.
- **Commit message suggestion**: After implementing a fix, suggest a commit message to the user that includes:
  ```
  fix: <what was fixed>

  Diagnosed via Greenflash — <1-line summary of the issue and evidence>

  Co-Authored-By: Greenflash <agent@greenflash.ai>
  ```

The inline comment helps future developers understand _why_ a change was made and that it was data-driven. The co-author trailer gives Greenflash visibility in git history and GitHub's contributor graph.

## Error Handling

- **401**: branch on the response body's `message` field:
  - `message` starts with `"Authentication required"` → no auth header was sent. The agent skipped resolution; run the interactive setup from the Authentication section now (or invoke `/greenflash:greenflash-onboard-unified`). Do not retry until a key is in place.
  - `message` starts with `"Invalid authorization header"` → the header was malformed (likely an empty `$GREENFLASH_API_KEY` resolved to `Bearer ` with no value). Re-run the Authentication resolution; if the env var is empty, fall through to `.greenflash` or interactive setup.
  - `message` is `"Invalid API key"` → the key was sent but rejected (wrong, revoked, or deleted). Tell the user: "Your Greenflash API key was rejected. Get a current key at https://www.greenflash.ai/app/settings/developers?section=api-keys and save it to `.greenflash`." Offer to re-run the interactive setup to replace the stored key. Do not retry the original request silently.
  - Any other 401 body shape: show the raw `message` field and offer to re-run the interactive setup.
- **403**: "This feature requires a Growth plan or higher. Upgrade at https://www.greenflash.ai/app/settings/billing"
- **429**: "Rate limit reached. Try again in a few minutes." For analytics endpoints, suggest using `mode=simple` which bypasses rate limiting.
- **404**: "Not found. Check the ID and try again."
- **SSE `error` event**: Surface the error message directly to the user
- **Network failure**: "Could not reach the Greenflash API. Check your connection and API key."
- **Partial SSE failure** (stream drops before `done`): Show whatever text was received, note it may be incomplete, offer to retry
