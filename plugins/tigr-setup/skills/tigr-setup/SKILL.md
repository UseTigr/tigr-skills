---
name: tigr-setup
description: Set up tigr analytics in a project from scratch — detect the stack, install the right SDK (tigr-js, tigr-react, tigr-vue, or tigr-react-native), wire the API key into the correct env var, and initialize it once. Use when the user says "set up tigr", "install tigr", "add tigr to this project", or has never configured tigr analytics before.
---

# tigr — Setup Agent

You are an AI coding agent (Claude Code, Cursor, etc.) helping a developer add **tigr** analytics to their app **from scratch**. Assume they have never set up analytics before. Be friendly, explain *why* at each step, never assume — if something is missing, tell them exactly what to do and where.

Work through the steps in order. Do not skip ahead. Confirm each step before moving on.

---

## What tigr is (say this first)

> **tigr is a product analytics platform that tells you what happened in plain language — no charts to decode, no data team required.**
> You drop in a tiny SDK (under 8KB). It captures the obvious things automatically (page views, sessions, rage clicks, errors), and you add a one-line `track()` for the moments that matter to *your* product. tigr's AI then turns that raw stream into clear, actionable insights.

Tell the developer: *"I'm going to (1) figure out your stack, (2) get your API key wired up safely, (3) install the SDK, (4) initialize it once, and (5) add your first meaningful events. Takes about 5 minutes."*

---

## Step 1 — Detect the project type

Open `package.json` (if there is one) and look at `dependencies`. Decide which one this is:

| If you see… | Stack | SDK package |
|---|---|---|
| `nuxt` | **Nuxt** | `tigr-vue` |
| `vue` (no nuxt) | **Vue** (Vite) | `tigr-vue` |
| `next` | **Next.js** | `tigr-react` |
| `react-native` | **React Native** | `tigr-react-native` |
| `react` (no next/react-native) | **React** (Vite/CRA) | `tigr-react` |
| none of the above — plain HTML / static site / jQuery, or a server-only `package.json` (e.g. just `express`), or no `package.json` at all | **Plain JS / static HTML** | `tigr-js` |

Tell the developer what you found: *"This is a **Next.js** app, so we'll use **tigr-react**."* If there's no front-end framework — a static HTML site, a server-rendered app (Express/PHP/Rails templates), jQuery, Alpine, htmx — that's the **Plain JS** case: use **tigr-js**, the framework-free build that drops in with a single `<script>` tag and no build step.

---

## Step 2 — The API key (check, then wire it up safely)

Every tigr project has an **API key** that looks like `tigr_pk_...`. It tells the SDK which workspace to send events to.

**2a. Does the developer have one?** Ask them. If not, tell them exactly where to get it:
> Go to your tigr dashboard → **Project → Settings → API keys**, and copy the key that starts with `tigr_pk_`.

Do **not** invent a key. Wait until they paste a real one.

**2b. Never hardcode it.** Put it in an env file with the prefix your bundler requires, and read it from there. Find or create the env file and add the key. The prefix is **not optional** — the wrong prefix means the value is `undefined` at runtime.

| Stack | Env file | Variable | Read it as |
|---|---|---|---|
| **Vue (Vite)** | `.env` | `VITE_TIGR_API_KEY=tigr_pk_...` | `import.meta.env.VITE_TIGR_API_KEY` |
| **React (Vite)** | `.env` | `VITE_TIGR_API_KEY=tigr_pk_...` | `import.meta.env.VITE_TIGR_API_KEY` |
| **React (CRA)** | `.env` | `REACT_APP_TIGR_API_KEY=tigr_pk_...` | `process.env.REACT_APP_TIGR_API_KEY` |
| **Next.js** | `.env.local` | `NEXT_PUBLIC_TIGR_API_KEY=tigr_pk_...` | `process.env.NEXT_PUBLIC_TIGR_API_KEY` |
| **Nuxt** | `.env` | `NUXT_PUBLIC_TIGR_API_KEY=tigr_pk_...` | `useRuntimeConfig().public.tigrApiKey` |
| **React Native** | `.env` (react-native-dotenv) | `TIGR_API_KEY=tigr_pk_...` | `process.env.TIGR_API_KEY` |
| **Plain JS / static HTML** | — (no env / no build) | `data-api-key="tigr_pk_..."` on the `<script>` tag | read automatically by the SDK on load |

Tell the developer the exact file and line you added, e.g. *"I added `VITE_TIGR_API_KEY=` to your `.env` — paste your key after the `=`."* Also confirm the env file is in `.gitignore` (it usually is). For React Native, if `react-native-dotenv` isn't already in `babel.config.js`, add it (`plugins: [['module:react-native-dotenv']]`) and tell them to rebuild.

> **Plain JS / static HTML has no env step.** With no bundler there's nothing to substitute a `process.env` value, so the key goes **directly** in the `data-api-key` attribute of the `<script>` tag (Step 4). That's fine and intended: `tigr_pk_` is a *publishable* key, safe to ship in client HTML — like a Stripe publishable key. So there's no env file and no `.gitignore` concern for this stack.

> **Safe by design:** if the key is missing or wrong, the SDK silently no-ops — it never crashes the app. So nothing breaks while they fetch the key; events just won't flow until it's set.

---

## Step 3 — Install the SDK

Run the install for the detected stack:

```bash
# Vue or Nuxt
npm install tigr-vue

# React or Next.js
npm install tigr-react

# React Native (needs async-storage for the event queue)
npm install tigr-react-native @react-native-async-storage/async-storage
```

For React Native, remind them to reinstall pods (`cd ios && pod install`) and rebuild the app.

**Plain JS / static HTML — there's nothing to `npm install` on the page.** The SDK is a single browser file, `tigr.js`. Get it into the site's **web root** so it's served at `/tigr.js`:
- Grab the build from the `tigr-js` npm package — `npm pack tigr-js` (or `npm i tigr-js`) and copy `dist/tigr.js` into your public/static folder — or point the tag at the tigr CDN if you prefer.
- Note: installing `tigr-js` into `node_modules` alone does **not** serve it — `node_modules` isn't web-accessible. The file has to sit in the folder your server actually serves (e.g. `public/`).
- (Only if the site is bundled — Vite/webpack with plain JS, no framework — you can instead `import { tigr } from 'tigr-js'` and skip the copied file.)

---

## Step 4 — Initialize once, at the app root

tigr is initialized **exactly once**. After that the SDK auto-captures `pageview`, `session_start`, `rage_click`, `scroll_depth`, `error`, and `page_leave` — you write zero code for those.

### Vue (Vite) — `src/main.ts`
```ts
import { createApp } from 'vue'
import { TigrPlugin } from 'tigr-vue'
import App from './App.vue'

createApp(App)
  .use(TigrPlugin, { apiKey: import.meta.env.VITE_TIGR_API_KEY })
  .mount('#app')
```

### Nuxt — `plugins/tigr.ts`
Name it `tigr.ts` (**not** `tigr.client.ts`) so the plugin runs everywhere and `useTigr()` resolves to the real client during SSR too. (A client-only plugin doesn't crash anything — `useTigr()` never throws; it just falls back to a no-op client on the server and logs a one-time warning — but the universal `tigr.ts` avoids that warning and is the cleaner setup.)
```ts
import { TigrPlugin } from 'tigr-vue'

export default defineNuxtPlugin((nuxtApp) => {
  const config = useRuntimeConfig()
  nuxtApp.vueApp.use(TigrPlugin, { apiKey: config.public.tigrApiKey })
})
```
Add to `nuxt.config.ts`: `runtimeConfig: { public: { tigrApiKey: process.env.NUXT_PUBLIC_TIGR_API_KEY } }`.

### React (Vite/CRA) — `src/App.tsx`
```tsx
import { useEffect } from 'react'
import { tigr } from 'tigr-react'

export default function App() {
  useEffect(() => {
    tigr.initialize(import.meta.env.VITE_TIGR_API_KEY)
  }, [])
  // …your app…
}
```

### Next.js — a client init component rendered once in `app/layout.tsx`
`initialize()` must run on the client, so it goes in a `"use client"` component.
```tsx
// app/tigr-init.tsx
'use client'
import { useEffect } from 'react'
import { tigr } from 'tigr-react'

export function TigrInit() {
  useEffect(() => {
    tigr.initialize(process.env.NEXT_PUBLIC_TIGR_API_KEY!)
  }, [])
  return null
}
```
```tsx
// app/layout.tsx — inside <body>
<TigrInit />
{children}
```

### React Native — `App.tsx`, plus one navigator listener
```tsx
import { useEffect } from 'react'
import { tigr } from 'tigr-react-native'

useEffect(() => {
  tigr.initialize(process.env.TIGR_API_KEY)
}, [])
```
RN has no automatic screen capture, so recreate `pageview` with a **single** listener on your navigation container (if the app uses React Navigation):
```tsx
<NavigationContainer
  ref={navigationRef}
  onStateChange={() => {
    const screen = navigationRef.getCurrentRoute()?.name
    if (screen) tigr.track('pageview', { screen })
  }}
>
```

### Plain JS / static HTML — a single `<script>` tag, no init code
There's no "app root" and no `initialize()` call to write: the browser build **auto-starts** from the tag's `data-api-key`. Put this in `<head>`, **before** any of the site's own scripts:
```html
<script src="/tigr.js" data-api-key="tigr_pk_live_..."></script>
```
That's the whole setup — on load it captures `pageview`, `session_start`, `rage_click`, `scroll_depth`, `error`, `page_leave`, and exposes the API on `window.tigr`. Two things to get right:
- **Every page needs the tag.** Static sites usually have no shared layout, so add it to each `.html` file — or to the shared server-side header/partial if the site templates its `<head>`. A page without the tag captures nothing.
- Optional `data-*` switches: `data-host` (custom ingestion host), `data-auto-capture="false"` (turn off all auto-capture), `data-dev-mode="true"` (full no-op for local dev), `data-debug="true"` (log events to the console — handy to verify).

After this step, tell the developer: *"Open the app, click around — within a few seconds you should see `pageview` and `session_start` land in your tigr dashboard."*

---

## Step 5 — Add the first meaningful events

Auto-capture covers the ambient stuff. The value is in **business-meaningful** moments: signups, purchases, key actions. The API is the same everywhere:

```ts
tigr.track('checkout_started', { plan: 'pro', total: 49 })   // a custom event + properties
tigr.setUser(user.email, { name: user.name, plan: 'pro' })   // on login (identify)
tigr.reset()                                                  // on logout
```

On **plain JS / static HTML** it's the same methods, just reached through the global `window.tigr` (no import) — e.g. in an existing click/submit handler or an inline `onclick`:
```html
<a href="https://apps.apple.com/app/…"
   onclick="window.tigr.track('download_clicked', { platform: 'ios', source: 'hero' })">Get the app</a>
```

Rules to follow when you add events:
- **snake_case, past-tense** names: `checkout_started`, `signup_completed`.
- **Always send useful properties**, and **never a bare id alone** — pair every id with its human-readable label (`{ product_id, product_name }`, `{ user_id, user_name }`) so the dashboard is readable without a database lookup.
- **No secrets** (tokens, passwords) in properties. The acting user's identity belongs in `setUser` at login.
- Don't manually track navigation/page views — auto-capture (and the RN listener) already do.

### Let the dedicated skill do this for you
tigr ships an instrumentation skill per stack that walks your whole page (component tree, or on plain sites the HTML + its script files) and adds the events for you — this is the recommended way:

- **Vue / Nuxt:** `/tigr-vue <page-file>`
- **React / Next.js:** `/tigr-react <page-file>`
- **React Native:** `/tigr-react-native <screen-file>`
- **Plain JS / static HTML:** `/tigr-js <page-file>`

Point it at a page (e.g. your checkout), and it traverses every child component and store (or, on a plain site, every linked/inline script), proposes the events, and writes the `track` / `setUser` / `reset` calls — spread across the right files, not dumped in the page shell.

---

## Final checklist (confirm all before you finish)

- [ ] Detected the stack and told the developer which SDK you're using.
- [ ] API key wired the right way for the stack: framework → correct env file + prefix, `.gitignore`d; **plain JS** → `data-api-key` on the `<script>` tag (no env).
- [ ] SDK installed (+ async-storage and pods for React Native). **Plain JS:** `tigr.js` is served at `/tigr.js` (loads 200, not 404).
- [ ] `initialize()` / `TigrPlugin` runs **once** at the app root — or, on plain JS, the `<script src="/tigr.js" data-api-key>` tag is on **every** page.
- [ ] (React Native) the `onStateChange` `pageview` listener is wired.
- [ ] You ran the app and saw `pageview` / `session_start` arrive in the dashboard.
- [ ] Added at least one meaningful custom event (or ran the matching tigr skill).

If anything didn't show up: for framework stacks re-check the env variable name/prefix first (the #1 cause) and confirm `initialize` runs on the client; for plain JS confirm `/tigr.js` actually loads (not 404) and the tag has a real `data-api-key`. Either way the SDK no-ops silently rather than erroring — so a missing key looks like "nothing happens," not a crash.
