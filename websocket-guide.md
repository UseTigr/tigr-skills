# Field Guide for WebSocket Applications

This document collects the things you should routinely think about, check, and guard against in
**any** application that uses WebSockets (or an abstraction like Socket.IO). It is written to be
project independent; the goal is a checklist you can open and apply item by item in any codebase.

Unlike HTTP, a WebSocket is a **persistent, stateful** connection. That means: the connection can
drop, go half open, accumulate resources on the server, and deliver messages out of order or
twice. With HTTP you think "request came in, I answered, done"; with WebSockets you write code
assuming "this connection will live for minutes and can break at any moment".

---

## 0. Mindset: why WebSockets are different

- **A connection is a resource.** Every open connection holds memory, timers, and room
  memberships on the server. These are not cleaned up automatically; if you do not clean them, they leak.
- **Never trust the client.** Every incoming message may be forged, incomplete, out of order, or malicious.
- **Drops are normal, not exceptional.** Mobile networks, wifi handoffs, tunnels, and sleep mode
  constantly break connections. A robust design treats dropping as expected.
- **The server is the single source of truth.** Client state may be stale at any moment; on
  reconnect, the server tells the truth.

---

## 1. Connection lifecycle

These events **must** be handled in every WS app:

- `connect` / `open`: connection established. Initial authentication and joining state happen here.
- `disconnect` / `close`: connection dropped. Log the `reason`, decide on resource cleanup.
- `reconnect` / re-`connect`: connected again. **State recovery** is triggered here.
- `error`: handshake or transport error.

**Guard:** `disconnect` is not the same as "the user left the room/game". Treating one as the
other is the most common mistake (see Sections 5 and 7).

---

## 2. Reconnection strategy

What to configure on the client:

- **Enable automatic reconnection.** (`reconnection: true`)
- **Exponential backoff + ceiling.** E.g. start at 500 ms, cap at 3-5 s. A fixed short interval
  hammers the server on a flaky network.
- **Attempt count.** Infinite retries are user friendly but drain battery/data; choose per product.
- **Jitter (random spread).** Do not let thousands of clients that dropped at once all return at
  the same instant ("thundering herd"). Add small randomness to the backoff.
- **Connection timeout.** If the handshake does not complete within a window (e.g. 20 s), fail
  instead of waiting forever.
- **Transport fallback.** WebSocket first, fall back to long polling if it cannot connect.
  Corporate proxies/firewalls sometimes block raw WS.

```
reconnection: true
reconnectionDelay: 500
reconnectionDelayMax: 3000
reconnectionAttempts: Infinity   // product decision
timeout: 20000
transports: ['websocket', 'polling']
```

---

## 3. State recovery (resume)

The most critical section. A client that drops and reconnects must be able to continue **from
where it left off**.

Pattern:

1. While the client is in an active session (room/match/channel), it stores that session's
   identity (e.g. `sessionId` + `userId`).
2. After every `connect` (first connect **and** every reconnect) the client automatically sends
   a `resume` message to the server: "I am this user, I was in this session".
3. If the session is still alive, the server: writes the client's new connection id (socket id)
   into the record, re-adds it to the room, and sends back the **full current state** (score,
   turn, time left, phase, etc.).
4. The client rebuilds its screen from this snapshot. It does not try to replay the intermediate
   events it missed one by one; a single moment of truth (snapshot) is enough.

**Guard:**
- Session state must live **on the server**; you cannot rely on the dropped client's memory to "continue".
- Resume must be idempotent: if resume arrives twice, state must not corrupt.
- On reconnect the socket id **changes.** Update all `socketId` references on the server with the
  new id, otherwise messages cannot reach the client (you send to a dead id).

---

## 4. Separate "drop" from "leave": grace period

Kicking a user out of a game because they entered a tunnel for 2 seconds is a bad experience.
Solution: a **grace period.**

- When the connection drops, do **not** end the session immediately. Start a timer (e.g. 10-30 s).
- If the user reconnects (resume) before the timer fires: cancel the timer, the session continues.
- If the timer fires and they still have not returned: **then** end it (award the opponent, drop
  from room, persist, etc.).

**Guard:**
- Tie the grace timer to the session/player and **cancel it** on reconnect. If you do not, it
  fires late and wrongly kicks a user who already came back.
- When the grace fires, re-verify "is this still the same dead connection?". The user may have
  returned on a new connection; do not let an old timer kill them.
- If the session **has not started yet**, a drop should usually be a silent cancel (no score/no
  record); if it has started, it should have consequences. Handle the two cases separately.

---

## 5. Event scoping: correlation IDs (the sneakiest source of bugs)

On a persistent connection, **a single socket can belong to multiple sessions/rooms at once**, or
may not have fully left an old room. So if an event coming from the server does not carry "which
session this belongs to", an **old session's event corrupts the new session.**

Classic symptom: you moved to a new session, but a message from the old session ("timer", "ended",
"opponent left") updates your screen incorrectly or throws you to the wrong screen.

**Rule:**
- Put a session identifier (`sessionId` / `roomId` / `matchId`) on **every** state message the
  server sends.
- The client **ignores** any message whose id does not match the session it is **currently** in.

```
socket.on('state-update', (msg) => {
  if (msg.sessionId !== currentSessionId) return; // not mine, drop it
  applyState(msg);
});
```

This one line guard eliminates the bulk of "post reconnect cross talk" bugs.

---

## 6. Server side resource cleanup (leak hunting)

Persistent connection = persistent resources. The following accumulate per session/room and must
be cleaned up **manually**:

- **Timers:** `setInterval` / `setTimeout` (game clock, grace, timeout). When a session ends or
  restarts, you **must** call `clearInterval`/`clearTimeout`.
- **Room memberships:** when a socket leaves a room, call `leave`. Only deleting the data
  structure does not remove the socket from the room; broadcasting to the old room still reaches it.
- **In memory maps:** delete finished sessions from structures like `Map<id, session>`.
- **Listeners:** remove the custom event handlers you bound to the connection on disconnect.

**The most classic leak:** starting a **new `setInterval`** for the same session without clearing
the old one. Two intervals decrement the same counter and you see weirdness like "the clock runs
twice as fast". Rule: **clear the old timer before starting a new one.**

```
function startTimer(session) {
  if (session.interval) clearInterval(session.interval); // clear first
  session.interval = setInterval(tick, 1000);
}
```

**Guard:** do not forget the "user left the screen but the connection is still open" case. If the
client does not notify the server when it changes screens, the session stays **alive** on the
server and its timer keeps running. Either the client sends an explicit "I am leaving" message
when it leaves, or the server closes that user's old session when they enter a new one.

---

## 7. Idempotency and duplicate protection

Messages can arrive again (retransmit, double tap, re-emit after reconnect).

- **Make writes idempotent.** If the same "submit" arrives twice, the result should be as if
  processed once.
- **Client side double submit lock.** Do not let the button/action re-trigger while it is processing.
- **Server side replay detection.** Reject the second operation that arrives in the same turn /
  with the same id (e.g. "this answer was already received").
- **Turn/sequence validation.** Silently drop operations that arrive "out of turn" or "while the
  session is not active".

---

## 8. Client side listener hygiene

- **Bind/cleanup balance.** Every listener you bind with `on(...)` when a screen/component mounts
  must be **removed** with `off(...)` when it unmounts. Otherwise the same event is handled
  multiple times (double update, double navigation).
- **Single connection (singleton) pattern.** Use one socket instance across the whole app; if
  each screen opens its own connection you get both resource waste and room/session confusion.
- **Stale closure trap.** When you set up a listener you "freeze" the current state; if you need
  the current value inside it, use a ref / a fresh read, otherwise you decide based on old values.

---

## 9. Authentication and authorization

- **Authenticate at the handshake.** Validate the token/session when the connection is
  established; reject if invalid.
- **If the token expires.** On a long lived connection the token can become invalid mid stream;
  consider periodic re-validation or refresh.
- **Authorize on every sensitive event.** Check "is this user a member of this room, is this user
  allowed to do this operation" on every message; checking once at connect time is not enough.
- **Room/channel authorization.** Do not let the client join any room it asks for; verify
  ownership/membership.

---

## 10. Zombie connections: heartbeat / ping pong

TCP does not always notice immediately that a connection has died ("half open" connection).
Result: the server thinks a disconnected client is alive and holds resources.

- **Enable ping/pong (heartbeat).** Most libraries have it; tune the interval and timeout values.
- **Drop unresponsive connections.** If no pong arrives within a window, terminate the connection
  and clean up resources.
- On mobile, sleep/background can silently kill a connection; heartbeat catches this.

---

## 11. Message delivery and ordering

- **Ordering guarantees are weak.** Especially during reconnect / multiple transports, message
  order can break. Do not write logic that depends on order; if possible each message should
  carry self contained full state.
- **There is no delivery guarantee.** For critical operations use an **ACK**: the client sends,
  the server processes and acknowledges; if no ack arrives the client retries (together with idempotency).
- **Snapshot > event replay.** Instead of replaying missed intermediate events one by one, sending
  the current full state (snapshot) to resync the client is more robust.

---

## 12. Network status and offline experience (client)

- **Network listener.** Watch the device's online/offline status; when offline show the user a
  clear warning ("connection lost, try again").
- **"Connected but no access" distinction.** A device can be on wifi but unable to reach the
  internet; check not only "is it connected" but "is it reachable".
- **Recover on return.** When the network comes back, auto reconnect and recover state
  (Section 3); return the user to the least disruptive point.

---

## 13. Scaling (multiple servers)

In memory state is fine on a single server; it breaks when you scale.

- **Sticky sessions or shared state.** Behind a load balancer the client must always land on the
  same server, or session state must live in a shared store (e.g. Redis).
- **Pub/Sub adapter.** Broadcasts to rooms must propagate across all server instances (e.g.
  Socket.IO Redis adapter). Otherwise a user on server A will not receive a message from server B.
- **In memory `Map` does not scale.** Global state that works on a single instance becomes
  inconsistent across many instances.

---

## 14. Abuse and security

- **Rate limiting.** Set a per client messages/second limit; block flood/spam.
- **Payload validation.** Validate the schema/types of every incoming message (empty, too large,
  unexpected fields).
- **Message size limit.** Prevent memory blowup from huge payloads.
- **Origin/CORS control.** Restrict which origins can connect.
- **TLS (wss).** Do not use unencrypted `ws://` in production.
- **Connection count limit.** Set a reasonable cap per IP/user (DoS surface).

---

## 15. Observability (essential for debugging)

- Log connect, disconnect (**with reason**), reconnect, and transport upgrade events.
- Track the number of active connections/sessions as a metric (leaks show up early).
- Put the session id + user id in logs so you can follow a single user's journey.
- Most problems happen in production, on bad networks; without good logs you cannot diagnose.

---

## 16. Testing

The happy path is easy; the real work is in the edge cases:

- Cut the network mid session and bring it back (does resume work?).
- Reconnect within the grace period and after it expires (are both behaviors correct?).
- The same user on two tabs/devices (what happens with a double session?).
- Rapidly leave and re-enter a screen (are the old session/timer cleaned up?).
- Send the same message twice (does idempotency hold?).
- Slow / packet dropping network simulation.

---

## Quick checklist (when adding any WS feature)

- [ ] Auto reconnect + backoff + jitter + timeout configured?
- [ ] Transport fallback (websocket -> polling) in place?
- [ ] After reconnect, automatic **resume** and a **full state snapshot** from the server?
- [ ] Is the changing connection id updated on the server after reconnect?
- [ ] Is there a **grace** for drops, and is the grace **cancelled** on reconnect?
- [ ] Does every state event from the server carry a **session id**, and does the client **ignore** foreign sessions?
- [ ] Are **all timers** cleared when a session ends/restarts (no double timer)?
- [ ] Is room membership released with `leave`, and the in memory map deleted?
- [ ] Is the "left the screen but still connected" case handled?
- [ ] Are client listeners removed with `off` (no double handling)?
- [ ] Are writes idempotent + a double submit lock present?
- [ ] Auth at handshake, authorization on sensitive events?
- [ ] Heartbeat/ping pong on, zombie connections dropped?
- [ ] Rate limit + payload validation + size limit present?
- [ ] For multi server: sticky sessions / pub-sub adapter planned?
- [ ] Connect/drop/reconnect logs and an active connection metric in place?
- [ ] Drop/grace/double session scenarios tested?

---

## The gist in one paragraph

Make the server the single source of truth and keep state there. Treat drops as normal: return the
user to where they left off with auto reconnect + resume + grace. Put a session id on every event
and let the client process only what belongs to its currently active session. Take ownership of
closing every timer, room membership, and listener you open; do not forget the "left the screen but
stayed connected" case. Never trust the client: validate, rate limit, make it idempotent.
