# tigr-skills

AI skills that install [tigr](https://usetigr.app) into your app and write your analytics events for you.

You point a skill at a page. It reads the actual code, walks every child component, store and hook, works out which user actions are worth measuring, shows you the plan, and writes the `track()` calls into the files that own the handlers. You review a diff instead of hand-writing instrumentation.

Works with any agent that reads skills: **Claude Code**, **Cursor**, and friends.

```
/tigr-react app/checkout/page.tsx

  Traversing 5 files
    app/checkout/page.tsx          layout only
    components/cart-summary.tsx    owns action
    components/pay-button.tsx      owns action
    hooks/use-checkout.ts          owns action
    hooks/use-auth.ts              owns action

  Proposed events
    pay-button.tsx    checkout_started    { plan, total }
    cart-summary.tsx  cart_item_removed   { product_id, product_name }
    use-checkout.ts   purchase_completed  { plan, total, currency }
    setUser() on login, reset() on logout in hooks/use-auth.ts
```

---

## Why this exists

Analytics dies at the instrumentation step. Everyone agrees events matter, then nobody wants to open forty components and decide what to name things. So the SDK goes in, six auto-captured events show up, and the interesting stuff, the conversions, never gets tracked.

These skills do that part. They know what tigr already captures for free, so they never double-track it, and they know what a useful event looks like, so you do not end up with a dashboard full of `button_clicked`.

---

## What is in here

| File | What it does |
| --- | --- |
| `tigr-setup-agent.md` | Start here if tigr is not installed yet. Detects your stack, wires the API key into the right env file with the right prefix, installs the SDK, initializes it once. |
| `tigr-react-skill.md` | Instruments a React or Next.js page. |
| `tigr-vue-skill.md` | Instruments a Vue or Nuxt page. |
| `tigr-react-native-skill.md` | Instruments a React Native screen, and wires the navigator listener that gives you screen views. |
| `tigr-js-skill.md` | Instruments a plain HTML, server-rendered, jQuery, Alpine or htmx page. No build step. |

---

## Install

Claude Code reads skills from a `.claude/skills/<name>/SKILL.md` folder. Per project:

```bash
git clone https://github.com/UseTigr/tigr-skills.git /tmp/tigr-skills
mkdir -p .claude/skills/tigr-react
cp /tmp/tigr-skills/tigr-react-skill.md .claude/skills/tigr-react/SKILL.md
```

To have them in every project, use `~/.claude/skills/` instead of `.claude/skills/`.

Cursor and other agents: point the agent at the markdown file directly, or paste its contents into your rules. These are plain markdown with no tooling behind them, so anything that can read a file can run them.

You can also fetch any skill straight off the site, no clone needed:

```
https://usetigr.app/skills/tigr-react-skill.md
https://usetigr.app/skills/tigr-vue-skill.md
https://usetigr.app/skills/tigr-react-native-skill.md
https://usetigr.app/skills/tigr-js-skill.md
https://usetigr.app/tigr-setup-agent.md
```

---

## Usage

**1. Get tigr into the project.** If you have never set it up, hand your agent the setup agent:

```
Read tigr-setup-agent.md and set tigr up in this project.
```

It asks for your `tigr_pk_...` key (tigr dashboard, Project → Settings → API keys), picks the right SDK, and initializes it once at the app root.

| Stack | Package | Env var |
| --- | --- | --- |
| Vue (Vite) | `tigr-vue` | `VITE_TIGR_API_KEY` |
| Nuxt | `tigr-vue` | `NUXT_PUBLIC_TIGR_API_KEY` |
| React (Vite) | `tigr-react` | `VITE_TIGR_API_KEY` |
| Next.js | `tigr-react` | `NEXT_PUBLIC_TIGR_API_KEY` |
| React Native | `tigr-react-native` | `TIGR_API_KEY` |
| Plain JS, static HTML | `tigr-js` | none, the key goes in `data-api-key` on the script tag |

**2. Point a skill at a page.**

```
/tigr-react app/checkout/page.tsx
/tigr-vue   app/pages/search.vue
/tigr-js    public/index.html
/tigr-react-native views/matchmaking.view.tsx
```

The skill traverses, proposes, waits for your OK, then writes.

---

## What you get for free

Install the SDK and these arrive with zero code. The skills know about them and will never add a manual event that duplicates one:

| Event | Fires on |
| --- | --- |
| `pageview` | every page load and SPA route change |
| `session_start` | a new session (30 minute window) |
| `rage_click` | 3 or more rapid clicks in the same spot |
| `scroll_depth` | how far down the page someone got |
| `error` | uncaught errors and unhandled rejections |
| `page_leave` | leaving the page or tab, also flushes the queue |
| `identify` | emitted when you call `setUser` |

React Native differs: there is no DOM, so the signals are `app_background`, `error` and `identify`, and `pageview` comes from a single listener on your navigation container that the skill wires for you.

---

## The rules the skills follow

These are the opinions baked into every skill, and the reason the output is worth keeping.

**Traverse exhaustively.** The page file is almost never where the interesting actions live. Buttons, list rows and form fields sit one to four levels down, and auth and purchases usually live in a store, hook or mutation. A skill that edits only the page shell has failed. Each one prints a file inventory before it proposes anything, so you can see it actually looked.

**Instrument intent, not noise.** Conversions and key actions get events. Ambient UI chatter does not. The test is whether a founder would ask "how often does this happen?"

**Never track navigation.** Every route change is already a `pageview`, so a manual event on a link click double-counts. The exception is when the intent matters beyond the view, like an outbound app store link, and then the event is named for the intent (`download_clicked`), not the navigation.

**Never send a bare id.** If a readable label is in scope it ships alongside the id: `{ product_id, product_name }`, not `{ product_id }`. Someone reading the dashboard should not need a database to know what happened.

**Names are snake_case and past tense.** `checkout_started`, `purchase_completed`, `search_submitted`.

**No secrets in properties.** No tokens, passwords or credentials. The acting user's identity belongs in `setUser` traits at login, set once, not repeated on every event.

**Analytics never breaks the app.** Every method is a safe no-op before init or with a missing key. Calls are fire and forget, batched, and never throw on a network failure.

---

## The SDKs

| Package | npm |
| --- | --- |
| `tigr-js` | https://www.npmjs.com/package/tigr-js |
| `tigr-react` | https://www.npmjs.com/package/tigr-react |
| `tigr-vue` | https://www.npmjs.com/package/tigr-vue |
| `tigr-react-native` | https://www.npmjs.com/package/tigr-react-native |

All open source under [github.com/UseTigr](https://github.com/UseTigr).

---

## What tigr does with the events

tigr is product analytics that tells you what happened in plain language. Once events flow, the AI reads the stream and writes you a short briefing instead of handing you another dashboard to decode:

> "Checkout got more traffic this week, but nobody finished. Every session that dropped did it on the same step, right after an item was removed, and mostly on the pro plan."

No charts to decode, no data team required. [See it in action](https://usetigr.app/stories/analytics-in-one-command).

---

## Links

- [Docs](https://usetigr.app/docs)
- [SDK reference](https://usetigr.app/reference)
- [Why tigr](https://usetigr.app/why)
- [Start free](https://usetigr.app/register)

## Contributing

Issues and pull requests welcome. If a skill missed an obvious event in your codebase, or invented one it should not have, that is worth an issue. The traversal rules are the product, and real projects are how they get sharper.
