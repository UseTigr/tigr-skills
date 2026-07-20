---
name: tigr-vue
description: Analyze a given Vue/Nuxt page and add tigr custom analytics events (track / setUser / reset) where they matter. Use when the user says "tigr-vue <file>", "instrument this page", "add tigr events here", "where should I track events", or asks to add custom analytics/events to a page using the tigr-vue SDK.
---

# tigr — Page Instrumentation

**Goal:** Take a target page, work out which *meaningful* user actions deserve a custom event, explain the plan, then add the `track()` / `setUser()` / `reset()` calls using the `tigr-vue` SDK.

**Your role:** A pragmatic analytics engineer. You instrument business-meaningful moments — not noise. You never duplicate what the SDK already captures automatically, and you never break SSR.

The user gives you a page (a file path, or "this page"). If none is given, ask which page.

---

## Step 0 — Prerequisites (do this first, once per project)

1. **Confirm `tigr-vue` is installed.** Check `package.json` dependencies. If missing, stop and tell the user to install it (`yarn add tigr-vue` / `npm i tigr-vue`) and set up the plugin first.

2. **Locate the Nuxt/Vue plugin** that installs `TigrPlugin` (usually `app/plugins/tigr*.ts` in Nuxt, or `main.ts` in plain Vue).

3. **SSR — prefer a universal plugin.** `useTigr()` reads the client via Vue `inject()`. If the Nuxt plugin is **client-only** (`tigr.client.ts`), the provide never runs during SSR — but this does **not** crash the page: `useTigr()` is written to never throw. It calls `inject(TIGR_KEY, null)` and, when nothing is provided, warns once and returns a shared **no-op client** (also a no-op on the server). So a client-only plugin just means server-rendered `<script setup>` gets the no-op fallback plus a console warning — not a 500.
   - The SDK is SSR-safe by design (the client is a no-op on the server anyway).
   - **Recommended:** make the plugin universal — rename `app/plugins/tigr.client.ts` → `app/plugins/tigr.ts`. Now the real provide runs everywhere, so `useTigr()` resolves to the injected client (a no-op on the server) with no fallback warning. Do this before instrumenting SSR pages, and tell the user you did.
   - Plain Vue (no SSR) is unaffected.

---

## Step 1 — Analyze the page

A page is just the root of a tree. The real buttons and forms almost always live several levels down in child components, so you **must walk the whole subtree — recursively — to the leaf components that own the handlers.** Do not stop at the page file or the first level. The event belongs in the component that owns the action, never in a thin wrapper that only lays out children.

**How to traverse (do this exhaustively):**

1. Read the page file. Note every component tag it renders and every `<NuxtLink>` / `<a>` / `<button>` / `<form>`.
2. For each child component tag, **resolve it to its file and read it too**, then repeat for *its* children — depth-first, until you reach components with no further child components (the leaves). Keep going; a 4-level deep button is common.
3. **Resolving Nuxt auto-imported tags → files:** a tag is PascalCase of a kebab-case filename, and nested folders become a prefix. So `<UserMenu />` → `components/user-menu.vue`, `<DashboardLiveFeed />` → `components/dashboard-live-feed.vue` or `components/dashboard/live-feed.vue`. If unsure, `grep -ri "name-fragment" app/components` to locate it. Don't guess — open the actual file.
4. Also follow logic that isn't a component: imported **composables** (`use-auth.ts`, `use-cart.ts`), stores, and util handlers — auth/login/logout and conversions often live there, not in the template.
5. Keep a running map: `component → interactive actions it owns → handler name`. That map is what you instrument.

Skip re-reading shared primitives (a generic `<BaseButton>`, `<Icon>`) once you've seen them — instrument at the **call site** that gives the action meaning (the parent that says "this button submits the search"), not inside the reusable primitive.

**This traversal is mandatory, and it is the entire point of the skill — not a nicety.** Detailed, product-wide events are the differentiator; a thin layer on the page shell is worthless. The page file is almost never where the interesting actions live — the buttons, list rows, form fields, and menu items that own the handlers sit one to four levels down in child components and in composables/stores. **If you finish having edited only the page files and not their child components and composables, you have failed the task. A page that renders six components and uses two composables means you open and consider all eight files, not one.**

Before you propose anything (Step 3), you MUST produce and show a **Component Inventory** of everything you opened and the actions each owns:

```
Visited (page → children → composables/stores):
  • pages/checkout.vue            → (layout only)
  • components/cart-summary.vue   → remove item, apply coupon
  • components/pay-button.vue     → submit payment
  • composables/use-cart.ts       → addItem / clearCart
Leaf components that own actions: pay-button.vue, cart-summary.vue
```

Do not advance to Step 2 until every child component and every imported composable/store reachable from the page has been opened and listed. If the inventory has one line, you have not traversed — go back.

From the full tree, build the list of every user-initiated action: button clicks, form submits, key link clicks, search/filter submissions, auth (login/logout), purchases/conversions, downloads, shares, plan changes, media plays, etc.

---

## Step 2 — Decide what deserves an event

**Already auto-captured by the SDK — DO NOT add manual events for these:**

| Auto event | Covers |
|---|---|
| `pageview` | every page **and SPA route change** — fired automatically by the router hook |
| `session_start` | new session |
| `rage_click` | 3+ rapid clicks on the same spot |
| `scroll_depth` | how far down the page |
| `error` | uncaught JS errors |
| `page_leave` | leaving the page/tab |

So **never** add `track()` for generic clicks, scrolls, errors, or — importantly — **navigation / page views / route changes.** On web, every route change (including SPA `<NuxtLink>` / `router.push` transitions) is already a `pageview`. Do **not** add a manual event on `<NuxtLink>` clicks, route watchers, `onMounted` of a page, or `router.afterEach` just to record that a page was seen — that double-counts.

The only time a navigation link gets an event is when the *action* matters beyond the page view — e.g. an outbound store link (`download_clicked`) or a CTA whose intent you want to measure — and even then the event names the **intent** (`download_clicked`), not the navigation. Plain in-app nav: leave it to auto-capture.

**What IS worth a custom event** — business-meaningful intent & conversion:

- Conversions: `signup_completed`, `purchase_completed`, `subscription_started`
- Intent / key actions: `plate_searched`, `download_clicked`, `demo_requested`, `filter_applied`, `share_clicked`
- Feature usage that tells a product story: `report_generated`, `file_exported`, `plan_upgraded`
- Auth → use `setUser` / `reset`, not `track` (see Step 4)

Rule of thumb: *if a founder would ask "how often does this happen?", it's an event.* If it's ambient UI noise, skip it.

---

## Step 3 — Propose before you write

Present a short, scannable plan and get a quick OK (the user invoked this skill to add events, so don't over-ask):

The plan lists one line **per event**, and every line names the **file → handler** it lands in — so events spread across the child components you traversed, not all in the page file:

```
Page: app/pages/search.vue   (traversed 4 files — see Component Inventory)
Proposed events (file → handler → event):
  • components/search-form.vue   → submit plate form    → plate_searched    { plate_format, source }
  • components/result-card.vue   → click "View report"  → report_opened     { plate, vehicle_make }
  • components/download-cta.vue  → click "Get the app"  → download_clicked  { platform }
Auth: setUser() on login in composables/use-auth.ts, reset() on logout
Skipping: generic nav clicks, page views, scroll (all auto-captured)
```

If every proposed event points at the same file, you have not traversed — go back to Step 1.

Keep event names **snake_case**, `noun_action` past-tense style (`plate_searched`, `download_clicked`).

**An event without its data is half-useful — always send the relevant data as properties.** When you instrument a handler, look at what's already in scope (the argument, the reactive refs, the selected item, the form values) and pass the meaningful fields as properties so the event answers "what exactly happened?":

- search → `{ query, result_count, filter }`
- add to cart / purchase → `{ product_id, product_name, price, currency, quantity, plan }`
- download → `{ platform }` (ios | android)
- filter/sort → `{ filter, value }`

Rules for properties:
- **Never send a bare id alone — pair every id with its human-readable label when one is in scope.** Someone reading the dashboard must be able to tell *who or what* the event is about without looking an id up in a database. If a `name` / `title` / `label` for the same entity is available at the call site (it usually is — in the route params, the selected row, the form values, or the store), send *both*: `{ product_id, product_name }`, `{ post_id, post_title }`, `{ user_id, user_name }`. Only send a lone id if no readable label exists in scope.
- Primitives only (string/number/boolean) — flatten objects, don't dump a whole entity; pick the fields that matter (`product.id` **and** `product.name`, not the entire `product`).
- **No secrets or auth material** in event props (tokens, passwords, raw credentials). But human-readable labels of the entity the action is *about* (product names, post titles, the plan being bought) ARE wanted — that legibility is the whole point. Keep the *acting* user's own identity in `setUser` traits at login (intentional, set once), not repeated on every event.
- Use stable snake_case keys (`result_count`, not `resultCount`).
- If a useful value isn't in scope at the call site, it's fine to read it from the same source the handler already uses (the store, the form values, route/query params) — don't fetch extra just to enrich an event.

---

## Step 4 — Implement

Match the surrounding code style. Minimal, self-contained, no new abstractions.

**Get the client (once, in `<script setup>`):**
```ts
import { useTigr } from 'tigr-vue'
const tigr = useTigr()
```

**Track a custom event in the handler:**
```ts
function onSearch(plate: string) {
  // …existing logic…
  tigr.track('plate_searched', { plate_format: detectFormat(plate), source: 'hero' })
}
```

**Identify on login / forget on logout** (instrument the auth composable/handler):
```ts
const tigr = useTigr()
const login = (user) => tigr.setUser(user.id, { name: user.name, email: user.email, plan: user.plan })
const logout = () => tigr.reset()
```

Notes:
- `setUser(userId, traits?)` and `identify(...)` are the same; use `setUser`.
- Call `track` inside the real event handler, after (or alongside) the existing logic — never block the user action on it (it's fire-and-forget; the SDK batches and never throws on network failure).
- `useTigr()` must be called at the top level of `setup` (it uses `inject`), not inside a handler.
- Don't add `tigr.flush()` unless there's a specific unload case the SDK doesn't already handle.

---

## Step 5 — Verify

- No syntax/type errors: run the project's typecheck/lint if available, or boot `dev` briefly and confirm the page still renders (HTTP 200) with no console errors.
- Confirm `useTigr()` is imported and called in `setup`, and the plugin is universal (Step 0.3) so SSR pages don't throw.
- Summarize what you added (events + props + locations) and what you deliberately skipped.

---

## Guardrails

- Production apps: change only what's needed; never touch unrelated code. Don't commit/push unless asked.
- One event per meaningful action — no double-firing (e.g., don't track a click that auto-capture + your event both cover).
- If the page has no meaningful custom events (pure content/marketing page already covered by auto-capture), say so plainly instead of inventing events.
