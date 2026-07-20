---
name: tigr-react
description: Analyze a given React / Next.js page or component and add tigr custom analytics events (track / setUser / reset) where they matter. Use when the user says "tigr-react <file>", "instrument this page", "add tigr events here", "where should I track events", or asks to add custom analytics/events to a React or Next.js screen using the tigr-react SDK.
---

# tigr — React / Next.js Instrumentation

**Goal:** Take a target page or component, work out which *meaningful* user actions deserve a custom event, explain the plan, then add the `track()` / `setUser()` / `reset()` calls using the `tigr-react` SDK.

**Your role:** A pragmatic analytics engineer. You instrument business-meaningful moments — not noise. You never duplicate what the SDK already captures automatically, you never break SSR, and analytics must never crash the app.

The user gives you a page/component (a file path, or "this page"). If none is given, ask which one.

---

## Step 0 — Prerequisites (do this first, once per project)

1. **Confirm `tigr-react` is installed.** Check `package.json` dependencies. If missing, stop and tell the user to install it (`npm i tigr-react` / `yarn add tigr-react`).

2. **Confirm the SDK is initialized once at app root.** The shipped API is the zero-boilerplate global — `tigr.initialize(apiKey)` called once on the client, exactly like `OneSignal.initialize`. **There is no provider and no hook to wire** (`tigr-react` exports `tigr`, plus `createTigr` / `TigrClient` for advanced use). Always instrument through the `tigr` global.
   - **Next.js (App Router):** `initialize()` must run in a **Client Component** (`"use client"`) inside a `useEffect`. The clean pattern is a tiny init component rendered once in `app/layout.tsx`:
     ```tsx
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
     // app/layout.tsx
     <body>
       <TigrInit />
       {children}
     </body>
     ```
   - **Plain React (Vite / CRA):** call it once in the root component's `useEffect` (usually `App.tsx`):
     ```tsx
     useEffect(() => {
       tigr.initialize(import.meta.env.VITE_TIGR_API_KEY) // Vite; CRA: process.env.REACT_APP_TIGR_API_KEY
     }, [])
     ```
   - If this call is missing, add it (or tell the user) before instrumenting anything. The API key comes from env (`NEXT_PUBLIC_*` / `VITE_*` / `REACT_APP_*`) — **never hardcode it**.
   - **Safe by design:** the underlying client is a no-op on the server (importing the module never touches the DOM at import time), and any `tigr.*` method called *before* `initialize()` is a silent no-op. Calls never crash the host app or break SSR.

3. **Pageview is automatic — do NOT wire anything for navigation.** Unlike React Native, the web SDK captures route changes for you: it monkey-patches `history.pushState` / `replaceState` and listens for `popstate`, so **every** SPA navigation (Next.js `<Link>` / router, React Router, plain history) fires a `pageview` with no router coupling and no per-page wiring. The single `initialize()` is the whole setup.

> The SDK ships a single global — `tigr`. There is no `<TigrProvider>` / `useTigr()` to set up (README examples that show them predate the shipped API). Always instrument through the `tigr` global.

---

## Step 1 — Analyze the page/component

A page is just the root of a tree. The real buttons and forms almost always live several levels down in child components, so you **must walk the whole subtree — recursively — to the leaf components that own the handlers.** Do not stop at the page file or the first level. The event belongs in the component that owns the action, never in a thin wrapper that only lays out children.

**How to traverse (do this exhaustively):**

1. Read the page/component file. Note every child component it renders and every `<button>` / `onClick` / `<a>` / `<Link>` / `<form>` / `onSubmit`.
2. **React components are explicit imports** — no auto-import. For each child component, follow its `import` statement to the file and read it too, then repeat for *its* children — depth-first, until you reach the leaves that own the `onClick` / `onSubmit` handlers. A 4-level-deep button is common.
3. Also follow logic that isn't a component: **stores** (zustand `use-*-store.ts`, Redux slices), **hooks** (`use-auth.ts`), **react-query / SWR mutations**, server actions, and util handlers — auth (login/logout), purchases, and conversions very often live in a store, hook, or mutation, not in the page's JSX.
4. Keep a running map: `component/store → user actions it owns → handler name`. That map is what you instrument.

Skip re-reading shared primitives (a generic `<Button>`, `<Icon>`) once you've seen them — instrument at the **call site** that gives the action meaning (the page that says "this button starts checkout"), not inside the reusable primitive.

**This traversal is mandatory, and it is the entire point of the skill — not a nicety.** Detailed, product-wide events are the differentiator; a thin layer on the page shell is worthless. The page file is almost never where the interesting actions live — the buttons, list rows, form fields, and menu items that own the `onClick`/`onSubmit` handlers sit one to four levels down in child components and in stores/hooks/mutations. **If you finish having edited only the page/route files and not their child components and stores, you have failed the task. A page that renders six components and reads two stores means you open and consider all eight files, not one.**

Before you propose anything (Step 3), you MUST produce and show a **Component Inventory** of everything you opened and the actions each owns:

```
Visited (page → children → stores/hooks/mutations):
  • app/checkout/page.tsx        → (layout only)
  • components/cart-summary.tsx  → remove item, apply coupon
  • components/pay-button.tsx    → submit payment
  • store/use-cart-store.ts      → addItem / clearCart
Leaf components that own actions: pay-button.tsx, cart-summary.tsx
```

Do not advance to Step 2 until every child component and every imported store/hook/mutation reachable from the page has been opened and listed. If the inventory has one line, you have not traversed — go back.

From the full tree, build the list of every user-initiated action: button clicks, form submits, key link intents, search/filter submits, auth (login/logout), purchases/conversions, downloads, shares, plan changes, media plays, etc.

---

## Step 2 — Decide what deserves an event

**Already auto-captured by the SDK — DO NOT add manual events for these** (all on by default; toggled via `autoCapture`):

| Auto event | Covers |
|---|---|
| `pageview` | every page **and SPA route change** — fired automatically by the history patch (Step 0.3) |
| `rage_click` | 3+ rapid clicks on the same spot |
| `scroll_depth` | how far down the page |
| `error` | uncaught JS errors |
| `page_leave` | leaving the page/tab |
| `identify` | emitted when you call `setUser` / `identify` |

So **never** add `track()` for generic clicks, scrolls, errors, page-leave, or — importantly — **navigation / page views / route changes.** Every route change (including SPA `<Link>` / `router.push` / React Router transitions) is already a `pageview`. Do **not** add a manual event on a `<Link>` click, a route change effect, a page's `useEffect`, or a router listener just to record that a page was seen — that double-counts.

The only time a navigating action also gets a custom event is when the *intent* matters beyond the page view — e.g. an outbound store link (`download_clicked`) or a CTA whose intent you want to measure — and even then the event names the **intent** (`download_clicked`), not the navigation. Plain in-app nav: leave it to auto-capture.

**What IS worth a custom event** — business-meaningful intent & conversion:

- Conversions: `signup_completed`, `purchase_completed`, `subscription_started`
- Intent / key actions: `checkout_started`, `download_clicked`, `demo_requested`, `filter_applied`, `share_clicked`
- Feature usage that tells a product story: `report_generated`, `file_exported`, `plan_upgraded`
- Auth → use `setUser` / `reset`, not `track` (see Step 4)

Rule of thumb: *if a founder would ask "how often does this happen?", it's an event.* If it's ambient UI noise, skip it.

---

## Step 3 — Propose before you write

Present a short, scannable plan and get a quick OK (the user invoked this skill to add events, so don't over-ask):

The plan lists one line **per event**, and every line names the **file → handler** it lands in — so events spread across the child components you traversed, not all in the page file:

```
Page: app/checkout/page.tsx   (traversed 5 files — see Component Inventory)
Proposed events (file → handler → event):
  • components/pay-button.tsx    → onClick "Buy"           → checkout_started   { plan, total }
  • components/cart-summary.tsx  → onClick "Remove"        → cart_item_removed  { product_id, product_name }
  • hooks/use-checkout.ts        → onSuccess mutation      → purchase_completed { plan, total, currency }
Auth: setUser() on login in hooks/use-auth.ts, reset() on logout
Skipping: nav / page views (auto pageview), clicks/scroll/errors/page-leave (auto)
```

If every proposed event points at the same file, you have not traversed — go back to Step 1.

Keep event names **snake_case**, `noun_action` past-tense style (`checkout_started`, `purchase_completed`).

**An event without its data is half-useful — always send the relevant data as properties.** When you instrument a handler, look at what's already in scope (the argument, props, state, the selected item, the form values) and pass the meaningful fields as properties so the event answers "what exactly happened?":

- search → `{ query, result_count, filter }`
- add to cart / purchase → `{ product_id, product_name, price, currency, quantity, plan }`
- download → `{ platform }` (ios | android)
- filter/sort → `{ filter, value }`

Rules for properties:
- **Never send a bare id alone — pair every id with its human-readable label when one is in scope.** Someone reading the dashboard must be able to tell *who or what* the event is about without looking an id up in a database. If a `name` / `title` / `label` for the same entity is available at the call site (it usually is — in the route params, the selected row, the mutation variables, or the store), send *both*: `{ product_id, product_name }`, `{ post_id, post_title }`, `{ user_id, user_name }`. Only send a lone id if no readable label exists in scope.
- Primitives only (string/number/boolean) — flatten objects, don't dump a whole entity; pick the fields that matter (`product.id` **and** `product.name`, not the entire `product`).
- **No secrets or auth material** in event props (tokens, passwords, raw credentials). But human-readable labels of the entity the action is *about* (product names, post titles, the team being joined) ARE wanted — that legibility is the whole point. Keep the *acting* user's own identity in `setUser` traits at login (intentional, set once), not repeated on every event.
- Use stable snake_case keys (`result_count`, not `resultCount`).
- If a useful value isn't in scope at the call site, read it from the same source the handler already uses (the store, the mutation variables, route/query params) — don't fetch extra just to enrich an event.

---

## Step 4 — Implement

Match the surrounding code style. Minimal, self-contained, no new abstractions.

**Import the global (no hook, no provider — works anywhere, including inside stores, mutations, server actions, and outside React):**
```ts
import { tigr } from 'tigr-react'
```

**Track a custom event in the handler:**
```tsx
function onCheckout() {
  // …existing logic…
  tigr.track('checkout_started', { plan: 'pro', total })
}
```

**Identify on login / forget on logout** — instrument the store/hook/mutation that owns auth, alongside the existing login/logout logic:
```ts
async function login(credentials) {
  const user = await api.login(credentials)
  tigr.setUser(user.id, { name: user.name, plan: user.plan }) // traits = profile fields, NOT PII you don't need
  // …
}

function logout() {
  tigr.reset()
  // …
}
```

Notes:
- `setUser(userId, traits?)` and `identify(...)` are the same; use `setUser`.
- The `tigr` global works **outside React too** — a zustand store action, a react-query mutation, a Next.js server-side helper, a util — because it's a module singleton, not a hook. That's why it can be called from anywhere with no `useTigr()` wiring.
- Call `track` inside the real handler, after (or alongside) the existing logic — never block the user action on it (it's fire-and-forget; the SDK batches and never throws on network failure).
- Don't add `tigr.flush()` — the SDK already flushes on `page_leave`. Only add a manual flush for a specific case the SDK doesn't cover.

---

## Step 5 — Verify

- No syntax/type errors: run the project's typecheck/lint if available (`npm run typecheck` / `tsc --noEmit`), or boot `dev` briefly and confirm the page still renders with no console errors.
- Confirm `tigr` is imported where used and `tigr.initialize()` exists once at app root on the client (Step 0.2) — in Next.js App Router, inside a `"use client"` component.
- Summarize what you added (events + props + locations) and what you deliberately skipped.

---

## Guardrails

- Production apps: change only what's needed; never touch unrelated code. Don't commit/push unless asked.
- One event per meaningful action — no double-firing.
- Pageview is automatic (Step 0.3) — **never** add events on plain navigation / `<Link>` clicks / route changes; auto-capture already records every page view.
- If the page has no meaningful custom events (pure content/marketing page already covered by auto-capture), say so plainly instead of inventing events.
