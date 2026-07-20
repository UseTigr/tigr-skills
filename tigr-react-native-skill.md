---
name: tigr-react-native
description: Analyze a given React Native screen/component and add tigr custom analytics events (track / setUser / reset) where they matter. Use when the user says "tigr-react-native <file>", "instrument this screen", "add tigr events here", "where should I track events", or asks to add custom analytics/events to a React Native screen using the tigr-react-native SDK.
---

# tigr — React Native Instrumentation

**Goal:** Take a target screen, work out which *meaningful* user actions deserve a custom event, explain the plan, then add the `track()` / `setUser()` / `reset()` calls using the `tigr-react-native` SDK.

**Your role:** A pragmatic analytics engineer. You instrument business-meaningful moments — not noise. You never duplicate what the SDK already captures automatically, and analytics must never crash the app.

The user gives you a screen (a file path, or "this screen"). If none is given, ask which one.

---

## Step 0 — Prerequisites (do this first, once per project)

1. **Confirm `tigr-react-native` is installed.** Check `package.json` dependencies. If missing, stop and tell the user to install it (`yarn add tigr-react-native` / `npm i tigr-react-native`).

2. **Confirm the SDK is initialized once at app root.** The recommended setup is the zero-boilerplate global — `tigr.initialize(apiKey)` called once in the root component's `useEffect` (usually `App.tsx`), exactly like `OneSignal.initialize`. No provider, no hook, no singleton wiring.
   ```ts
   import { tigr } from 'tigr-react-native'

   useEffect(() => {
     tigr.initialize(TIGR_API_KEY)   // apiKey from @env
   }, [])
   ```
   - If this call is missing, add it (or tell the user) before instrumenting anything. The API key is read from env (`@env` / react-native-dotenv) — never hardcode it.
   - **Safe by design:** any `tigr.*` method called *before* `initialize()` is a silent no-op, and the SDK no-ops (instead of throwing) outside a real RN runtime or when `devMode: true`. Calls never crash the host app.

3. **Wire `pageview` on the navigator (if the project has one) — do this once, here.** RN has no automatic screen capture, so if the app uses React Navigation we recreate the web's auto-`pageview` ourselves with a **single** listener on the navigation container. Add an `onStateChange` to `<NavigationContainer>` (the project usually already has a `navigationRef`):
   ```tsx
   <NavigationContainer
     ref={navigationRef}
     onStateChange={() => {
       const screen = navigationRef.getCurrentRoute()?.name
       if (screen) tigr.track('pageview', { screen })
     }}
   >
   ```
   This makes every screen change a `pageview { screen }`, exactly like the web SDK fires on every route change — so the dashboard treats RN screens and web pages the same. Add this once during setup. If there's no navigator (single-screen app), skip it. Don't scatter per-screen `useEffect` tracking — one listener is the whole job.

4. **No SSR, no plugin.** Unlike the web/Nuxt SDK there's nothing to make "universal" — RN has no server render. The only things that must exist are the single `initialize()` and (if there's a navigator) the single `onStateChange` `pageview` above.

> The SDK ships a single global — `tigr` (plus `createTigr` / `TigrClient` for advanced use). There is no provider or hook; always instrument through the `tigr` global.

---

## Step 1 — Analyze the screen

A screen is just the root of a tree. The real buttons and forms almost always live several levels down in child components, so you **must walk the whole subtree — recursively — to the leaf components that own the handlers.** Do not stop at the screen file or the first level. The event belongs in the component that owns the action, never in a thin wrapper that only lays out children.

**How to traverse (do this exhaustively):**

1. Read the screen file. Note every child component it renders and every `<Pressable>` / `<TouchableOpacity>` / `<Button>` / `onPress` / form submit / `onSubmitEditing`.
2. **RN components are explicit imports** — no auto-import. For each child component tag, follow its `import` statement to the file and read it too, then repeat for *its* children — depth-first, until you reach the leaves that own the `onPress` handlers. A 4-level-deep button is common.
3. Also follow logic that isn't a component: **zustand stores** (`use-*-store.ts`), hooks (`use-auth.ts`), react-query mutations, and util handlers — auth (login/logout), purchases, and conversions very often live in a store or mutation, not in the screen's JSX. (In TypeClash, e.g., login/logout live in `use-player-store.ts`.)
4. Keep a running map: `component/store → user actions it owns → handler name`. That map is what you instrument.

Skip re-reading shared primitives (a generic `<AppButton>`, `<Icon>`) once you've seen them — instrument at the **call site** that gives the action meaning (the screen that says "this button starts matchmaking"), not inside the reusable primitive.

**This traversal is mandatory, and it is the entire point of the skill — not a nicety.** Detailed, product-wide events are the differentiator; a thin layer on the screen shell is worthless. The screen/`*.view.tsx` file is almost never where the interesting actions live — the buttons, list rows, form fields, and menu items that own the `onPress`/submit handlers sit one to four levels down in child components and in stores/hooks/mutations. **If you finish having edited only the screen/`view` files and not their child components and stores, you have failed the task. A screen that renders six components and reads two stores means you open and consider all eight files, not one.**

Before you propose anything (Step 3), you MUST produce and show a **Component Inventory** of everything you opened and the actions each owns:

```
Visited (screen → children → stores/hooks/mutations):
  • views/block-user.view.tsx     → submit (block user)
  • components/user-row.tsx       → onPress row → open profile
  • components/block-form.tsx     → duration select, reason input, submit
  • store/user.state.ts           → setUser / logout
Leaf components that own actions: block-form.tsx, user-row.tsx
```

Do not advance to Step 2 until every child component and every imported store/hook/mutation reachable from the screen has been opened and listed. If the inventory has one line, you have not traversed — go back.

From the full tree, build the list of every user-initiated action: button presses, form submits, key navigation intents, search/filter submits, auth (login/logout), purchases/IAP, game/feature actions, shares, etc.

---

## Step 2 — Decide what deserves an event

**Already auto-captured by the SDK — DO NOT add manual events for these:**

The two auto-capture signals are toggled via `autoCapture` (`{ lifecycle, error }`) in the config — both on by default.

| Auto event | Covers |
|---|---|
| `app_background` | app sent to background (the `lifecycle` signal — mobile equivalent of web's `page_leave`) |
| `error` | uncaught JS errors (the `error` signal, via `ErrorUtils`) |
| `identify` | emitted when you call `setUser` / `identify` |
| `pageview` | **once you wire the navigator listener (Step 0.3)** — every screen change. The SDK does not fire this on its own; the `onStateChange` listener does. |

So **never** add `track()` for generic errors, app background/foreground, or session bookkeeping.

**Navigation is already covered — do NOT track `navigate` calls.** Once the `pageview` listener from Step 0.3 is in place, every screen change is recorded automatically, exactly like the web SDK. So treat navigation like the web skill does: **never** add a `track()` on a `navigation.navigate(...)` / `navigationRef.navigate(...)` call, a "Go to X" button whose only job is navigation, or a screen's `useEffect`, just to record that a screen was seen — the `pageview` listener already does it and a manual event would double-count.

The only time a navigating action also gets a custom event is when the *intent* matters beyond the screen view (e.g. `match_started` on a button that both starts a match and pushes the game screen) — and then the event names the **intent**, not the navigation. Plain "go to this screen" navigation: leave it to the `pageview` listener.

**What IS worth a custom event** — business-meaningful intent & conversion:

- Conversions: `signup_completed`, `purchase_completed`, `subscription_started`
- Intent / key actions: `match_started`, `game_finished`, `invite_sent`, `filter_applied`, `share_pressed`
- Feature usage that tells a product story: `word_cup_entered`, `friend_request_sent`, `settings_changed`
- Auth → use `setUser` / `reset`, not `track` (see Step 4)

Rule of thumb: *if a founder would ask "how often does this happen?", it's an event.* If it's ambient UI noise, skip it.

---

## Step 3 — Propose before you write

Present a short, scannable plan and get a quick OK (the user invoked this skill to add events, so don't over-ask):

The plan lists one line **per event**, and every line names the **file → handler** it lands in — so events spread across the child components you traversed, not all in the screen file:

```
Screen: views/matchmaking.view.tsx   (traversed 5 files — see Component Inventory)
Proposed events (file → handler → event):
  • components/find-opponent-button.tsx → onPress "Find opponent" → match_started   { mode, language }
  • components/match-result-card.tsx    → onPress "Rematch"        → rematch_pressed  { opponent_id, opponent_name }
  • store/use-player-store.ts           → result handler           → game_finished    { won, score, opponent_id, opponent_name, mode }
Auth: setUser() on login in store/use-player-store.ts, reset() on logout
Skipping: navigation / screen views (pageview listener), errors/background (auto)
```

If every proposed event points at the same file, you have not traversed — go back to Step 1.

Keep event names **snake_case**, `noun_action` past-tense style (`match_started`, `game_finished`).

**An event without its data is half-useful — always send the relevant data as properties.** When you instrument a handler, look at what's already in scope (the argument, store state, the selected item, the form values) and pass the meaningful fields as properties so the event answers "what exactly happened?":

- search → `{ query, result_count, filter }`
- purchase / IAP → `{ product_id, product_name, price, currency, plan }`
- game result → `{ won, score, opponent_id, opponent_name, mode }`
- filter/sort → `{ filter, value }`
- block / report / assign → `{ blocked_user_id, blocked_user_name, reason }`

Rules for properties:
- **Never send a bare id alone — pair every id with its human-readable label when one is in scope.** A panel reader must be able to tell *who or what* the event is about without looking an id up in a database. If a `name` / `title` / `fullname` / label for the same entity is available at the call site (it very often is — in route params, the selected item, the mutation variables, or the store), send *both*: `{ blocked_user_id, blocked_user_name }`, `{ job_id, job_title }`, `{ driver_id, driver_name }`. Only send a lone id if no readable label exists in scope — and prefer pulling it from the same source the handler already uses rather than leaving the event illegible.
- Primitives only (string/number/boolean) — flatten objects, don't dump a whole entity; pick the fields that matter (`opponent.id` **and** `opponent.name`, not the entire object).
- **No secrets or auth material** in event props (tokens, passwords, raw credentials). But human-readable labels of the entity the action is *about* (names, titles of the blocked user, the assigned driver, the purchased plan) ARE wanted — that legibility is the whole point. Keep the *acting* user's own identity in `setUser` traits at login (so it's intentional, not repeated on every event), not the label of the thing they acted on.
- Use stable snake_case keys (`result_count`, not `resultCount`).
- If a useful value isn't in scope at the call site, read it from the same source the handler already uses (the store, the mutation variables, route params) — don't fetch extra just to enrich an event.

---

## Step 4 — Implement

Match the surrounding code style. Minimal, self-contained, no new abstractions.

**Import the global (no hook, no provider — works anywhere, including inside zustand stores and outside React):**
```ts
import { tigr } from 'tigr-react-native'
```

**Track a custom event in the handler:**
```ts
function onFindOpponent() {
  // …existing logic…
  tigr.track('match_started', { mode: 'ranked', language })
}
```

**Identify on login / forget on logout** — instrument the store/hook that owns auth (in TypeClash this is `use-player-store.ts`, alongside `OneSignal.login` / `OneSignal.logout`):
```ts
setPlayer: (id, nickname, phoneId) => {
  OneSignal.login(id)
  tigr.setUser(id, { nickname })   // traits = profile fields, NOT PII you don't need
  // …
},
logout: () => {
  OneSignal.logout()
  tigr.reset()
  // …
},
```

Notes:
- `setUser(userId, traits?)` and `identify(...)` are the same; use `setUser`.
- The `tigr` global works **outside React too** — a zustand store action, a navigation listener, a util — because it's a module singleton, not a hook. That's why it can be called from anywhere.
- Call `track` inside the real handler, after (or alongside) the existing logic — never block the user action on it (it's fire-and-forget; the SDK batches and never throws on network failure).
- Don't add `tigr.flush()` — the SDK already flushes on `app_background`, which is the critical mobile moment. Only add a manual flush for a specific case the SDK doesn't cover.

---

## Step 5 — Verify

- No syntax/type errors: run the project's typecheck/lint if available (`yarn tsc --noEmit` / the project's typecheck script).
- Confirm `tigr` is imported where used and `tigr.initialize()` exists once at app root (Step 0.2).
- Summarize what you added (events + props + locations) and what you deliberately skipped.

---

## Guardrails

- Production apps: change only what's needed; never touch unrelated code. Don't commit/push unless asked.
- One event per meaningful action — no double-firing.
- Wire the navigator `pageview` listener once (Step 0.3); after that, **never** add events on plain navigation — the listener already records every screen change.
- If the screen has no meaningful custom events (pure layout/content already covered by auto-capture), say so plainly instead of inventing events.
