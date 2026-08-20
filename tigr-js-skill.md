---
name: tigr-js
description: Analyze a given plain-JavaScript / static HTML page (no framework, no build step) and add tigr custom analytics events (track / setUser / reset) where they matter. Use when the user says "tigr-js <file>", "instrument this page", "add tigr events here", "where should I track events", or asks to add custom analytics/events to a vanilla-JS, server-rendered, jQuery, or static HTML site using the tigr-js SDK.
---

# tigr — Plain JavaScript / Static HTML Instrumentation

**Goal:** Take a target page (an `.html` file or the JS that drives it), work out which *meaningful* user actions deserve a custom event, explain the plan, then add the `track()` / `setUser()` / `reset()` calls using the `tigr-js` SDK.

**Your role:** A pragmatic analytics engineer. You instrument business-meaningful moments — not noise. You never duplicate what the SDK already captures automatically, and analytics must never crash the host page.

The user gives you a page (a file path, or "this page"). If none is given, ask which one.

This is the **framework-free** SDK — for plain HTML sites, server-rendered pages (PHP/Express/Rails templates), jQuery, Alpine, htmx, or anything that isn't Vue/React/RN. There is **no build step, no components, no imports** — the SDK lives on `window.tigr`.

---

## Step 0 — Prerequisites (do this first, once per site)

1. **Confirm `tigr.js` is loaded.** Unlike the framework SDKs there's no `package.json` dependency to check — the browser build is a single `<script>` tag. Grep the site's HTML (all pages, or the shared header/template) for `tigr.js` / `data-api-key`. The install is a one-liner in `<head>`, **before** any of the site's own scripts:
   ```html
   <script src="/tigr.js" data-api-key="tigr_pk_live_..."></script>
   ```
   - On load it **auto-starts** from `data-api-key` and begins capturing pageviews, scroll depth, errors and page-leaves. No init code needed.
   - The file must be **served** by the site. On a static/Express site drop `dist/tigr.js` (from the `tigr-js` npm package, or the CDN) into the public web root so `/tigr.js` resolves. `npm i tigr-js` alone does **not** serve it — `node_modules` isn't web-accessible; it's for the module/bundler path only.
   - The API key comes from the `data-api-key` attribute — **never invent one**; if it's missing, tell the user to paste their `tigr_pk_...` key. Safe by design: a missing/wrong key silently no-ops, it never crashes the page.
   - `data-*` options: `data-host` (override ingestion host), `data-auto-capture="false"` (disable all auto-capture), `data-dev-mode="true"` (full no-op for local dev), `data-debug="true"` (console-log events).

2. **Multi-page sites: the tag must exist on EVERY page.** Static sites usually have no shared layout — each `.html` file is standalone. Confirm the `<script src="/tigr.js" ...>` is present in each page you care about (or in the shared server-side header/partial if the site templates its `<head>`). A page without the tag captures nothing. If the site has a single shared header include, one edit covers all pages; if not, each HTML file needs the tag.

3. **No SSR concern, no plugin, no hook.** The IIFE build is a no-op on the server anyway and only runs in the browser. The only thing that must exist is the single `<script>` tag per page. The global is always `window.tigr`; every method is a safe no-op before init or if the key is missing.

> The SDK ships a single browser global — `window.tigr` (with `initialize` / `track` / `setUser` / `identify` / `reset` / `flush`). For bundler/module setups it also exports `tigr` and `createTigr` / `TigrClient` from `'tigr-js'`, but on a plain HTML site you always instrument through `window.tigr`.

---

## Step 1 — Analyze the page

A plain-JS page has no component tree, but it still has a **structure you must walk exhaustively**: the HTML markup, its inline `<script>` blocks, **and every external JS file it loads**. The real handlers almost always live in a linked `.js` file or a script block further down the page — not next to the button. So you must follow every `<script src>` and read the handler code, never stop at the markup.

**How to traverse (do this exhaustively):**

1. Read the page `.html` file. Note every interactive element and how it's wired:
   - inline handlers: `onclick="..."`, `onsubmit="..."` attributes
   - `<form>` elements (their `action` / submit handler)
   - `<a>` / `<button>` — especially outbound links (app-store, download, external CTAs)
   - elements the JS grabs by `id` / `class` / `data-*` and binds later
2. **Follow every `<script src="...">` to its file and read it** — that's where `addEventListener('click', …)`, form-submit handlers, `fetch`/AJAX calls, jQuery `$('#x').on('click', …)`, Alpine `@click`, or htmx triggers actually live. Repeat for each linked JS file. Also read inline `<script>` blocks in full.
3. Also follow shared JS: a `main.js` / `app.js` / `analytics.js` / auth helper that multiple pages include — login/logout, cart, and conversions very often live in one shared script, not in the page.
4. Keep a running map: `file → user actions it owns → handler`. That map is what you instrument.

Skip re-reading generic helpers (a `$.ajax` wrapper, a toast util) once you've seen them — instrument at the **call site** that gives the action meaning (the handler that says "this submits the search"), not inside the reusable helper.

**This traversal is mandatory, and it is the entire point of the skill — not a nicety.** Detailed, product-wide events are the differentiator; a thin layer on one page's markup is worthless. The interesting actions (form submits, CTA clicks, list-row clicks, menu items) usually sit in a linked JS file or a script block, and shared actions (auth, cart) sit in a shared script. **If you finish having edited only the HTML markup and not the JS files that own the handlers, you have failed the task. A page that loads three script files and a form means you open and consider all of them, not just the `.html`.**

Before you propose anything (Step 3), you MUST produce and show a **File Inventory** of everything you opened and the actions each owns:

```
Visited (page → scripts → shared JS):
  • public/index.html          → hero CTA (download), inline script block
  • public/js/search.js        → submit search form
  • public/js/main.js          → mobile-menu toggle, outbound app-store links
  • public/js/auth.js          → login / logout
Files that own actions: search.js, auth.js, index.html (inline)
```

Do not advance to Step 2 until every linked/inline script reachable from the page has been opened and listed. If the inventory has one line, you have not traversed — go back.

From the full picture, build the list of every user-initiated action: button clicks, form submits, key link intents (downloads, outbound CTAs), search/filter submits, auth (login/logout), purchases/conversions, shares, plan changes, media plays, etc.

---

## Step 2 — Decide what deserves an event

**Already auto-captured by the SDK — DO NOT add manual events for these** (all on by default; toggled via `data-auto-capture`):

| Auto event | Covers |
|---|---|
| `pageview` | every page load **and** SPA/history navigation — fired automatically (initial load + a `history.pushState`/`replaceState` patch + `popstate`) |
| `session_start` | new session (30-min window) |
| `scroll_depth` | how far down the page |
| `error` | uncaught JS errors + unhandled promise rejections |
| `page_leave` | leaving the page/tab (also flushes) |
| `identify` | emitted when you call `setUser` / `identify` |

So **never** add `track()` for generic clicks, scrolls, errors, page-leave, or — importantly — **navigation / page views.** On a classic multi-page site every new page is a fresh document load → an automatic `pageview`. Do **not** add a manual event on a plain `<a href>` in-site link, an `onload`, or a router hook just to record that a page was seen — that double-counts.

The only time a navigating link also gets a custom event is when the *intent* matters beyond the page view — e.g. an outbound app-store / download link (`download_clicked`) or a CTA whose intent you want to measure — and even then the event names the **intent** (`download_clicked`), not the navigation. Plain in-site nav: leave it to auto-capture.

**What IS worth a custom event** — business-meaningful intent & conversion:

- Conversions: `signup_completed`, `purchase_completed`, `subscription_started`, `contact_submitted`
- Intent / key actions: `download_clicked`, `demo_requested`, `search_submitted`, `filter_applied`, `share_clicked`
- Feature usage that tells a product story: `video_played`, `pricing_toggled`, `faq_opened`
- Auth → use `setUser` / `reset`, not `track` (see Step 4)

Rule of thumb: *if a founder would ask "how often does this happen?", it's an event.* If it's ambient UI noise, skip it.

---

## Step 3 — Propose before you write

Present a short, scannable plan and get a quick OK (the user invoked this skill to add events, so don't over-ask):

The plan lists one line **per event**, and every line names the **file → handler** it lands in — so events spread across the JS files you traversed, not all in one page's markup:

```
Page: public/index.html   (traversed 4 files — see File Inventory)
Proposed events (file → handler → event):
  • public/js/search.js    → submit search form    → search_submitted  { query, result_count }
  • public/index.html      → hero "Get the app"     → download_clicked  { platform, source }
  • public/js/contact.js   → contact form submit    → contact_submitted { topic }
Auth: setUser() on login in public/js/auth.js, reset() on logout
Skipping: in-site nav / page views (auto pageview), clicks/scroll/errors/page-leave (auto)
```

If every proposed event points at the same file, you have not traversed — go back to Step 1.

Keep event names **snake_case**, `noun_action` past-tense style (`search_submitted`, `download_clicked`).

**An event without its data is half-useful — always send the relevant data as properties.** When you instrument a handler, look at what's already in scope (the form values, the clicked element's `data-*`, the query string, the item in the loop) and pass the meaningful fields as properties so the event answers "what exactly happened?":

- search → `{ query, result_count, filter }`
- add to cart / purchase → `{ product_id, product_name, price, currency, quantity, plan }`
- download → `{ platform }` (ios | android), plus a `source` when the same CTA appears in several places
- filter/sort → `{ filter, value }`

Rules for properties:
- **Never send a bare id alone — pair every id with its human-readable label when one is in scope.** Someone reading the dashboard must be able to tell *who or what* the event is about without a database lookup. If a `name` / `title` / `label` for the same entity is available at the call site (in the DOM `data-*`, the form values, or the query string), send *both*: `{ product_id, product_name }`, `{ post_id, post_title }`. Only send a lone id if no readable label exists in scope.
- Primitives only (string/number/boolean) — flatten, don't dump a whole object.
- **No secrets or auth material** in event props (tokens, passwords, raw credentials). Human-readable labels of the entity the action is *about* (product names, plan being bought) ARE wanted — that legibility is the whole point. Keep the *acting* user's own identity in `setUser` traits at login (set once), not repeated on every event.
- Use stable snake_case keys (`result_count`, not `resultCount`).
- If a useful value isn't in scope at the call site, read it from the same source the handler already uses (the form, the element's dataset, the URL) — don't fetch extra just to enrich an event.

---

## Step 4 — Implement

Match the surrounding code style. Minimal, self-contained, no new abstractions. Everything goes through `window.tigr` — no import, no init (the script tag already initialized it).

**Track a custom event in the existing handler:**
```js
document.querySelector('#search-form').addEventListener('submit', (e) => {
  // …existing logic…
  window.tigr.track('search_submitted', { query, result_count: results.length })
})
```

**Inline handler in the HTML (fine for simple CTAs):**
```html
<a href="https://apps.apple.com/app/…"
   onclick="window.tigr.track('download_clicked', { platform: 'ios', source: 'hero' })">
  Get the app
</a>
```

**Identify on login / forget on logout** — instrument the shared auth script alongside the existing login/logout logic:
```js
function onLoginSuccess(user) {
  window.tigr.setUser(user.id, { name: user.name, plan: user.plan }) // traits = profile fields, not PII you don't need
  // …
}
function logout() {
  window.tigr.reset()
  // …
}
```

Notes:
- `setUser(userId, traits?)` and `identify(...)` are the same; use `setUser`.
- `window.tigr` is **always defined** (the script tag sets it), and every method is a safe no-op before init / with a missing key — so you don't need `window.tigr && …` guards. (If you instrument a script that might run before `tigr.js` loads, use `window.tigr?.track(...)`.)
- Call `track` inside the real handler, after (or alongside) the existing logic — never block the user action on it (it's fire-and-forget; the SDK batches and never throws on network failure).
- Don't add `window.tigr.flush()` — the SDK already flushes on `page_leave`. Only add a manual flush for a specific case the SDK doesn't cover.
- **Module/bundler sites (not plain HTML):** import instead — `import { tigr } from 'tigr-js'`, call `tigr.initialize('tigr_pk_...')` once, then `tigr.track(...)`. Same method surface.

---

## Step 5 — Verify

- No syntax errors: open the page in a browser (or the project's dev server) and confirm it still renders with no console errors.
- Confirm the `<script src="/tigr.js" data-api-key="...">` tag is present (Step 0) on the pages you instrumented and that `/tigr.js` actually loads (200, not 404).
- With `data-debug="true"` you can watch events log to the console — a quick way to confirm your `track()` fires and `pageview` lands.
- Summarize what you added (events + props + locations) and what you deliberately skipped.

---

## Guardrails

- Production sites: change only what's needed; never touch unrelated code. Don't commit/push unless asked.
- One event per meaningful action — no double-firing.
- Pageview is automatic — **never** add events on plain in-site navigation / `<a href>` clicks / page loads; auto-capture already records every page view.
- If the page has no meaningful custom events (pure content/marketing page already covered by auto-capture), say so plainly instead of inventing events.
