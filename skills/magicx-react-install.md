# Install AI Autocomplete (React) — agent instructions

You are an AI coding agent installing **AI Autocomplete** into the user's React app. Follow these steps in order. Do not skip the "Hand off to the user" section — those steps are not yours to perform.

Reference docs: https://ai-autocomplete.com/docs/react/getting-started

AI Autocomplete uses two key types:

- **Public keys** (`pk_v1_...`) — safe to ship in client bundles. Sent as a Bearer token directly from the browser. Use this for the basic install path below.
- **Secret keys** (`sk_v1_...`) — server-only. Your server exchanges a secret key for a short-lived access token (`at_v1_...`) that the SDK consumes. Recommended for production. See the "Production: secret key + access token" section at the end.

---

## Step 0 — Read the SDK README

Before any code changes, run:

```
npm view @magicx-eng/ai-autocomplete-react readme
```

Read the output end-to-end. It documents the full prop surface, both auth modes, the CSS variables, and the stable `data-aia-*` selector hooks — you'll need it in Steps 2 and 5. If you can't run shell commands, fetch `https://unpkg.com/@magicx-eng/ai-autocomplete-react/README.md` instead — same content.

## Step 1 — Ask the user which integration tier

The SDK ships two tiers. **Stop, ask the user, and wait for their answer.** Do not pick on their behalf.

> "AI Autocomplete ships two integration tiers — which do you want?
>
> **Tier 1 — Full component (`<AIAutocomplete />`)**
> Drop-in. The SDK owns the textarea, pills, dropdown, and all state. You pass props (auth, callbacks, theme) and you're done. This is the right choice for most apps.
>
> **Tier 2 — Headless (`useAIAutocomplete()` + `<AIAutocompleteDropdown />`)**
> You own the input element and layout. The hook returns state, actions, and spread props (`inputProps`, `dropdownProps`). Pick this if you have a custom input (rich text, search bar, existing form field) or need full control over rendering.
>
> Which one — full component or headless?"

If the user says "you decide" or seems unsure, recommend according to their current product.

## Step 2 — Install and wire up the SDK

Sanity-check the project: there should be a `package.json` at the root. If not, ask the user where their app lives. Note the framework if it's obvious (Vite, Next.js, etc.) — that only matters for the `"use client"` directive on Next.js App Router.

Install the SDK with the project's package manager:

- pnpm: `pnpm add @magicx-eng/ai-autocomplete-react`
- yarn: `yarn add @magicx-eng/ai-autocomplete-react`
- npm: `npm install @magicx-eng/ai-autocomplete-react`
- bun: `bun add @magicx-eng/ai-autocomplete-react`

If `react` and `react-dom` are not already in `dependencies`, install them too.

Then add the integration code for the tier the user picked. **Hardcode the public key as the literal string `"PASTE_PUBLIC_KEY_HERE"`** — leave that placeholder in the code. The user will replace it after Step 3. Do **not** invent a key, and do **not** create a `.env`/`.env.local` entry for it. Once the install is verified end-to-end it's the user's call how to manage the key long-term (env var, config service, secret manager).

If you don't know where to mount the component, ask the user before guessing.

**Tier 2 rule: always call `reset()` from your own submit handler.** Tier 1 (`<AIAutocomplete />`) does this automatically. In Tier 2 the consumer owns the textarea and the submit flow — don't pass `onSubmit` to `useAIAutocomplete()`. Wire up submit however you like (a form `onSubmit`, a button `onClick`, a custom Enter-key handler) and at the end of that handler call the `reset` returned by `useAIAutocomplete()`. It clears the input and rotates the per-session `session_id` so the server's history stays accurate; skipping it degrades suggestion quality on subsequent submissions.

### Tier 1 — Full component

```tsx
import { AIAutocomplete } from "@magicx-eng/ai-autocomplete-react";

export function MyAutocomplete() {
  return (
    <AIAutocomplete
      apiConfig={{
        apiKey: "PASTE_PUBLIC_KEY_HERE",
      }}
      mode="dark"
      onSubmit={(result) => {
        console.log(result.query);
        console.log(result.completed_params);
      }}
    />
  );
}
```

The `mode` prop accepts `"light"` or `"dark"` and defaults to `"dark"`. Match the surrounding page — light theme → `mode="light"`; dark or unsure → leave it as `"dark"` (or omit).

### Tier 2 — Headless

```tsx
import {
  useAIAutocomplete,
  AIAutocompleteDropdown,
} from "@magicx-eng/ai-autocomplete-react";

export function MyAutocomplete() {
  const { inputProps, dropdownProps, completedParams, isLoading, error, reset } =
    useAIAutocomplete({
      apiConfig: { apiKey: "PASTE_PUBLIC_KEY_HERE" },
    });

  const handleSubmit = () => {
    console.log(inputProps.value);
    console.log(completedParams);
    reset();
  };

  return (
    <div>
      <textarea {...inputProps} />
      <AIAutocompleteDropdown {...dropdownProps} />
      <button type="button" onClick={handleSubmit}>Submit</button>
    </div>
  );
}
```

If the user already has a custom input element (rich text editor, search bar, etc.), spread `inputProps` onto that element instead of a fresh `<textarea />`. The README's `useAIAutocomplete` and `<AIAutocompleteDropdown />` sections cover the full state/action surface and the rendering knobs (`mode`, `pillPlacement`, `optionsPosition`) that move from the wrapping component down to the dropdown component in this tier.

### Next.js note

For Next.js App Router, mark the parent file `"use client"`.

## Step 3 — Hand off to the user (you cannot do this)

After your code changes are complete, **stop** and print the following to the user verbatim — including the bullets and URLs. These are browser tasks; the agent cannot perform them.

> **Two browser steps you need to do before this works:**
>
> 1. Go to **https://ai-autocomplete.com/edit** and create your product. This configures what your autocomplete will suggest (the prompts, parameter types, and option lists).
> 2. Go to **https://ai-autocomplete.com/account/keys**, create a **public key**, copy it, and paste it in place of `PASTE_PUBLIC_KEY_HERE` in the component. The public key starts with `pk_v1_`.
>
> Once both are done, reload the page (or restart the dev server if hot reload didn't pick it up). Reply when you've confirmed it works (or share what's failing).

Do not pretend you can open URLs, create the product, or generate the key on the user's behalf. Wait for their reply before moving to Step 4.

## Step 4 — Verify it works

Once the user replies that they've pasted the key, walk them through verification:

1. Run the dev server (or reload the page).
2. Open the page where you mounted the autocomplete and confirm it renders without console errors.
3. If the network tab shows **401 Unauthorized** on `/api/suggest`, the key is wrong or it isn't a public key — ask the user to re-copy it from `/account/keys` and confirm it starts with `pk_v1_`.
4. If you see **404** or "no suggestions returned", the product likely wasn't created yet — ask the user to confirm they finished Step 3.1.

Wait for an explicit "yes, it works" before moving on.

## Step 5 — Offer to customize the styling

Once the user confirms the autocomplete works, ask:

> "Want to customize the look? The SDK exposes CSS variables and stable `data-aia-*` selector hooks for everything from pill colors to the submit button. I can apply changes against your existing palette/theme — tell me what you want different (e.g. 'match my brand primary', 'tighten the pill padding', 'switch to a solid dropdown'), or say 'looks fine' to wrap up."

If the user wants changes, refer to the **CSS Variables** and **Selector Hooks** tables in the SDK README you read in Step 0.

Apply overrides on a class you pass via `className` on `<AIAutocomplete />` (or the wrapping element in Tier 2). All defaults use `:where()` (zero specificity) so any class you write wins. For per-mode overrides, target `[data-mode="light"]` / `[data-mode="dark"]` on the same element.

For changes the variables don't cover, target the `data-aia-*` selector hooks (e.g. `[data-aia-dropdown]`, `[data-aia-pill]`, `[data-aia-textarea]`) — those are the stable public surface. Don't target hashed CSS-module class names.

If the user says "looks fine" or doesn't want changes, you're done. Mention briefly that for production they may want to switch to the access-token flow below, but don't build it unless asked.

---

## Production: secret key + access token (recommended)

Public keys work, but for production browser apps we recommend the access-token flow:

- The **secret key** (`sk_v1_...`) lives **only on the user's backend server** — never in the browser bundle, never in client-side env vars (no `VITE_`/`NEXT_PUBLIC_`/`REACT_APP_` prefix), never in a frontend `.env`.
- The user's **backend** exchanges the secret key for a short-lived **access token** (`at_v1_...`) by calling AI Autocomplete.
- The **frontend** SDK fetches that access token from a route on the user's *own* backend and uses it. The browser never touches the secret key.

Only build this once the basic install above is verified, and confirm with the user before adding server code. **If the project has no backend (e.g. a static site or pure SPA with no server runtime), STOP and tell the user — you cannot do the secret-key flow without a server. Stay on public keys.**

### A. Add server-side env vars

These live in the user's **backend** environment (Vercel/Railway/Fly server env, Docker secrets, host shell — wherever the server runs). They must NOT have `VITE_`, `NEXT_PUBLIC_`, or `REACT_APP_` prefixes — those would expose them to the browser.

```
MAGICX_SECRET_KEY=sk_v1_...
MAGICX_PRODUCT_ID=<the product UUID from /edit>
```

The product UUID is visible in the URL of the product editor at `/edit`. Ask the user to copy it.

### B. Add the token-exchange route on the backend

> **This code runs on the user's backend server, not in the browser.** The secret key is read from server-side `process.env` and used in an outbound `fetch` from the server. The browser never sees it.

The backend hits `POST https://api.ai-autocomplete.com/api/auth/token` with the secret key as a Bearer token and the `product_id` in the body. The endpoint returns `{ access_token, expires_at, token_type }`. Forward that response to the browser unchanged.

Next.js App Router example — `app/api/ac-token/route.ts` (Route Handlers run on the server, not the client):

```ts
// SERVER-ONLY
export async function GET() {
  const res = await fetch("https://api.ai-autocomplete.com/api/auth/token", {
    method: "POST",
    headers: {
      Authorization: `Bearer ${process.env.MAGICX_SECRET_KEY}`,
      "Content-Type": "application/json",
    },
    body: JSON.stringify({ product_id: process.env.MAGICX_PRODUCT_ID }),
  });

  if (!res.ok) {
    // 401 with error.code === "INVALID_SECRET_KEY" means a config problem
    // on the user's end — don't surface it to the browser as 401.
    return new Response("Token exchange failed", { status: 502 });
  }

  return Response.json(await res.json());
}
```

For Pages Router, use `pages/api/ac-token.ts`. For Express/Fastify/Hono/serverless, port the same shape: same upstream URL, same body, same response passthrough — always on the **backend** side of the codebase.

### C. Switch the SDK to access-token mode (browser-side)

> **This code runs in the browser.** It contains no secret key — only the path of the user's own backend route from Step B.

```tsx
// BROWSER
<AIAutocomplete
  apiConfig={{
    type: "accessToken",
    getAccessToken: async () => {
      const res = await fetch("/api/ac-token");
      const { access_token, expires_at } = await res.json();
      return { accessToken: access_token, expiresAt: expires_at };
    },
  }}
  onSubmit={handleSubmit}
/>
```

### D. Remove the public key

Once the access-token flow is verified, drop the hardcoded public key (or whatever the user moved it into) from the component — you don't need it anymore.

Full docs: https://ai-autocomplete.com/docs/react/authentication
