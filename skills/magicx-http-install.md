# Install AI Autocomplete (HTTP API) — agent instructions

You are an AI coding agent integrating **AI Autocomplete** into the user's app over raw HTTP — no SDK. Follow these steps in order. Do not skip the "Hand off to the user" section — those steps are not yours to perform.

Reference docs: https://ai-autocomplete.com/docs/http/getting-started

AI Autocomplete uses three credential types over a single endpoint (`POST https://api.ai-autocomplete.com/api/suggest`):

- **Public key** (`pk_v1_...`) — safe to ship in client bundles. Sent as `Authorization: Bearer pk_v1_...` from the client. Use this for the basic install path below.
- **Secret key** (`sk_v1_...`) — server-only. Used by the client's backend to mint short-lived access tokens, or sent directly as `Authorization: Bearer sk_v1_...` from a trusted server-to-server context. Never ship this in client code.
- **Access token** (`at_v1_...`) — short-lived. Minted from a secret key on the user's backend and forwarded to the client. Recommended for production browser/mobile apps. See the "Production: secret key + access token" section at the end.

---

## Step 0 — Read the HTTP reference

Before any code changes, fetch and read the HTTP API docs end-to-end:

- https://ai-autocomplete.com/docs/http/getting-started — request/response shape, the iteration loop, placeholder construction
- https://ai-autocomplete.com/docs/http/api-reference — field-by-field schema
- https://ai-autocomplete.com/docs/http/authentication — key types, token-mint endpoint
- https://ai-autocomplete.com/docs/http/advanced — debouncing, cancellation, stale-response handling, PII masking, session lifecycle

You'll need all of this. The HTTP API is small (one endpoint), but the loop semantics and the running `completed_params` list need to be exactly right.

## Step 1 — Ask the user where the call originates

The right credential and integration shape depend on where the request runs. **Stop, ask the user, and wait for their answer.** Do not pick on their behalf.

> "Where will calls to /api/suggest originate?
>
> **A. Browser or mobile client, directly** — public key in client code. Fast to set up; fine for prototypes and unscoped public credentials.
>
> **B. Browser or mobile client, with server-minted access tokens** — your backend holds a secret key, mints short-lived `at_v1_*` tokens, and the client uses those. Recommended for production.
>
> **C. Server-to-server** — your backend calls /api/suggest directly using a secret key. For batch jobs, back-office tools, or any non-interactive context.
>
> Which one — A, B, or C?"

If the user says "you decide" or seems unsure, recommend A for a quick prototype, or B if they're already on a production-style stack with a backend.

## Step 2 — Ask the language/runtime

Identify the language and HTTP client. Common targets:

- **JavaScript / TypeScript** (browser fetch, Node fetch, Axios)
- **Python** (requests, httpx)
- **Swift / iOS** (URLSession)
- **Kotlin / Android** (OkHttp, Ktor)
- **Go** (net/http)
- **Ruby** (Net::HTTP, Faraday)
- **Other** — use the standard HTTP client for that runtime

Ask the user if it's not obvious from the codebase.

## Step 3 — Write the integration code

**Two representations of the query — keep both in sync.** The query exists as two strings:

- **display** — what the user sees and edits, with the real chosen values: `Create a email`
- **raw_query** — what you send to the API, with each picked value replaced by its `{{TYPE_N}}` placeholder: `Create a {{TYPE_1}}`

`completed_params` is the bridge: each entry maps a `placeholder` to its `text`. Build `raw_query` from the display by swapping each picked value for its placeholder; reconstruct the display by swapping each placeholder back to its `text`. **The server only ever sees `raw_query` + `completed_params` — the display string is yours to maintain on the client.** A common mistake is to keep a single string and send the literal text in `raw_query`; then the placeholder named in `completed_params` matches nothing, the server can't resolve the param, and it never registers as completed (so suggestions stop advancing).

Generate code that:

1. **Tracks the autocomplete state** — the **display** text the user sees, the parallel **raw_query** with `{{TYPE_N}}` placeholders, and a running list of `completed_params` (each with `placeholder`, `type`, and `text`).
2. **Mints a `session_id`** (UUIDv4) when the autocomplete starts and **reuses** it across keystrokes within one session.
3. **Sends a POST** to `https://api.ai-autocomplete.com/api/suggest` on each keystroke (debounced — see Step 4) with:
   - `Authorization: Bearer <key>` (use the literal placeholder `PASTE_PUBLIC_KEY_HERE` for path A — the user will replace it in Step 5)
   - `Content-Type: application/json`
   - Body: `{ "data": { "raw_query": "<text with placeholders>", "completed_params": [...] }, "meta": { "request_id": "<uuidv4>", "request_at": "<ISO 8601>", "session_id": "<uuidv4>" } }`
4. **Constructs placeholder tokens** when the user picks a suggestion option. Format: `{{TYPE_N}}` — the param's `type` field uppercased (spaces → underscores), with a 1-based counter per type. First `goal` pick → `{{GOAL_1}}`; a second → `{{GOAL_2}}`; first `contact` → `{{CONTACT_1}}`. Names must be unique within `raw_query` because the server replaces them by string match.
5. **Substitutes** the chosen text in `raw_query` with the placeholder token, and **appends** an entry to `completed_params` recording `{ placeholder, type, text }`.
6. **Handles the response loop** — `data.suggestions[]` is what to show the user next. Each suggestion has a `type`, a `text` label, and (often) `options[]` the user can tap.

For path B or C, also generate the token-mint code (see the Production section below).

If you don't know where to wire up the autocomplete in the user's UI, ask before guessing.

### Reference body shape (any language)

```json
{
  "data": {
    "raw_query": "Create a {{TYPE_1}}",
    "completed_params": [
      { "placeholder": "{{TYPE_1}}", "type": "type", "text": "email" }
    ]
  },
  "meta": {
    "request_id": "5a0d1c1e-1b3f-4a9e-8a4f-001a1c3b7d20",
    "request_at": "2026-05-26T18:30:00Z",
    "session_id": "9e5b7c0e-2a1b-4f8e-9c2d-3e4f5a6b7c8d"
  }
}
```

Required server-side: `meta.request_id` (non-empty) and `meta.request_at` (non-zero). Everything else is optional but recommended.

## Step 4 — Wire up the loop concerns

Autocomplete fires requests on every keystroke. Don't skip these — without them, the UX breaks under fast typing:

- **Debounce** keystrokes — wait briefly after the last keystroke before sending, to collapse bursts without feeling laggy. 100–200ms works well; **prefer 100ms** for a snappy, responsive feel, moving toward 200ms to trim request volume during fast typing.
- **Cancel in-flight** — when a newer keystroke fires, abort the previous request using whatever cancellation mechanism the language provides.
- **Discard stale responses** — track the `request_id` (or `request_at`) of the most recent call; when a response arrives, compare `meta.request_id` — if it doesn't match the latest, drop the response.

If the user picks a UI framework with built-in state management (React, Vue, SwiftUI), generate idiomatic code for that framework. Otherwise plain async/await with a debounce timer is fine.

## Step 5 — Hand off to the user (you cannot do this)

After your code changes are complete, **stop** and print the following to the user verbatim. These are browser tasks; the agent cannot perform them.

> **Two browser steps you need to do before this works:**
>
> 1. Go to **https://ai-autocomplete.com/edit** and create your product. This configures what your autocomplete will suggest (the prompts, parameter types, and option lists).
> 2. Go to **https://ai-autocomplete.com/account/keys**, create a key, copy it, and paste it in place of `PASTE_PUBLIC_KEY_HERE` (or wherever your code reads the credential).
>    - Path A → create a **public key** (starts with `pk_v1_`).
>    - Path B or C → create a **secret key** (starts with `sk_v1_`).
>
> Once both are done, restart the dev server or reload, and reply when you've confirmed it works (or share what's failing).

Do not pretend you can open URLs, create the product, or generate the key on the user's behalf. Wait for their reply before moving to Step 6.

## Step 6 — Verify it works

Once the user replies that they've pasted the key, walk them through verification:

1. Send the first request — `raw_query: ""` or `raw_query: "Create a"` with an empty `completed_params`. Should return suggestions including a `type` suggestion with options like `email`, `automation`, `insight`.
2. Confirm the response includes a `meta.session_id` (either echoed from your request or generated by the server).
3. If you see **401 Unauthorized**, the key is wrong or the wrong type — ask the user to re-copy from `/account/keys` and confirm the prefix matches the path they chose (`pk_v1_` for A, `sk_v1_` for B/C's server side).
4. If you see **400 Bad Request**, check that `meta.request_id` and `meta.request_at` are both populated and well-formed.
5. Simulate a pick: take an option from the first response (e.g. `"email"`), construct `{{TYPE_1}}`, send `raw_query: "Create a {{TYPE_1}}"` with a `completed_params` entry. Confirm the response shows new suggestions appropriate for the next state.

Wait for an explicit "yes, it works" before moving on.

---

## Production: secret key + access token (recommended)

Public keys work, but for production browser/mobile apps the access-token flow is safer:

- The **secret key** (`sk_v1_...`) lives **only on the user's backend server** — never in the browser bundle, never in client-side env vars, never in a frontend `.env`.
- The user's **backend** exchanges the secret key for a short-lived **access token** (`at_v1_...`) by calling AI Autocomplete's token endpoint.
- The **frontend** fetches that access token from a route on the user's *own* backend and sends it as `Authorization: Bearer at_v1_...` when calling `/api/suggest`. The browser never touches the secret key.

Only build this once the basic install above is verified, and confirm with the user before adding server code. **If the project has no backend (e.g. a static site or pure SPA with no server runtime), STOP and tell the user — you cannot do the secret-key flow without a server. Stay on public keys.**

### A. Add server-side env vars

These live in the user's **backend** environment (Vercel/Railway/Fly server env, Docker secrets, host shell — wherever the server runs). They must NOT have client-exposing prefixes like `VITE_`, `NEXT_PUBLIC_`, `REACT_APP_`, or `EXPO_PUBLIC_`.

```
MAGICX_SECRET_KEY=sk_v1_...
MAGICX_PRODUCT_ID=<the product UUID from /edit>
```

The product UUID is visible in the URL of the product editor at `/edit`. Ask the user to copy it.

### B. Add the token-exchange route on the backend

> **This code runs on the user's backend server, not in the browser.** The secret key is read from server-side environment variables and used in an outbound HTTP call. The browser never sees it.

The backend POSTs `https://api.ai-autocomplete.com/api/auth/token` with the secret key as a Bearer token and the `product_id` in the JSON body. The endpoint returns `{ "access_token": "at_v1_...", "expires_at": <unix-seconds>, "token_type": "Bearer" }`. Forward that response to the browser unchanged.

The route shape is the same regardless of language. In pseudocode:

```
GET /api/ac-token  →  on the user's server:

  res = POST https://api.ai-autocomplete.com/api/auth/token
        headers:
          Authorization: Bearer <env.MAGICX_SECRET_KEY>
          Content-Type: application/json
        body: { "product_id": "<env.MAGICX_PRODUCT_ID>" }

  if !res.ok: return 502 to the browser
  else:       return res.json() to the browser
```

Port this to Express/Fastify/Hono/Next.js Route Handlers/Django/Flask/FastAPI/Go net/http/Rails/Sinatra/whatever the user's backend is. Same upstream URL, same body, same response passthrough.

### C. Use the access token on the client

> **This code runs in the browser/mobile.** It contains no secret key — only the path of the user's own backend route from step B.

On each `/api/suggest` call:

1. Fetch the access token from your backend (`GET /api/ac-token`).
2. Cache `access_token` alongside `expires_at`. Reuse it until you're within ~30 seconds of `expires_at`, then re-fetch.
3. If multiple concurrent calls need a token, deduplicate so only one fetch is in flight at a time.
4. Send `Authorization: Bearer <access_token>` on `/api/suggest`.
5. On a 401 response, force-refresh the token once and retry the original request. If the second attempt also 401s, surface the error — don't loop.

### D. Remove the public key

Once the access-token flow is verified, drop the hardcoded public key from the client code — you don't need it anymore.

Full docs: https://ai-autocomplete.com/docs/http/authentication
