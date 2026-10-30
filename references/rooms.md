# Rooms

Live shared state inside a hosted service: a chat room, a presence list, a shared document or a game session is one room, reached by name from the fetch handler, with its own WebSocket connections, storage and alarm. Docs: https://docs.yard.sh/v1/platform/services/rooms.md (contract, storage, alarms, limits, allowance, deletion, the dashboard's Rooms page), https://docs.yard.sh/v1/platform/projects/allowances.md. Needs `.team_permissions.service_rooms`.

- A room is a named instance of a room type, a class `_service.js` exports. Every request for one name reaches the same instance, which handles one event at a time, sleeps when idle (connections stay open) and wakes on the next message, request or alarm. Instance fields do not survive a sleep; storage and connection attachments do.
- One conversation, game or document is one room; never funnel everything through one name. The fetch handler only picks the room by name and forwards; the room owns the connections and the live state. Lasting data (accounts, history) stays in `env.DB`, which a room reaches through its `env` when the service has `database_access`.

## Declaring

On the service entry in `.yard/settings.json`: `"rooms": [{ "type": "Room", "binding": "ROOMS" }]`. `type` is the exported class name (`^[A-Z][A-Za-z0-9_]{0,63}$`, at most 10 per service); `binding` (optional, default the type in upper snake case) is reached as `env.<binding>` and follows the secret-name grammar. A declared type with no exported class answers every request through its binding with a 500; `yard service check` warns. `yard service init <name> --realtime` scaffolds a working broadcast `Room` plus a WebSocket client.

## Contract

| Method | Called when |
| --- | --- |
| `constructor(ctx, env)` | the room wakes; `ctx` is the room's context (connections, storage, alarm), `env` the service's (secrets, `DB`) |
| `fetch(request)` | the handler forwarded a request; return a `Response`, or a 101 carrying the client end of a `WebSocketPair` |
| `webSocketMessage(ws, message)` | a held connection sent a `string` or `ArrayBuffer` |
| `webSocketClose(ws, code, reason, wasClean)`, `webSocketError(ws, error)` | a held connection closed or failed |
| `alarm()` | the time set with `ctx.storage.setAlarm(ts)` arrived; runs with no connections open; retried if it throws |

```js
export default {
  async fetch(request, env) {
    const url = new URL(request.url);
    if (url.pathname === "/ws") {
      const room = env.ROOMS.get(env.ROOMS.idFromName(url.searchParams.get("room") ?? "lobby"));
      return room.fetch(request);
    }
    return env.ASSETS.fetch(request);
  },
};

export class Room {
  constructor(ctx, env) { this.ctx = ctx; this.env = env; }
  async fetch(request) {
    if (request.headers.get("Upgrade") !== "websocket") return new Response("Expected a WebSocket", { status: 426 });
    const user = request.headers.get("X-Yard-User-Id") ?? request.headers.get("X-Yard-Guest-Id") ?? "anonymous";
    const [client, server] = Object.values(new WebSocketPair());
    this.ctx.acceptWebSocket(server, [user]);           // hand the socket to Yard; the room may sleep between messages
    server.serializeAttachment({ user, joined: Date.now() });
    return new Response(null, { status: 101, webSocket: client });
  }
  async webSocketMessage(ws, message) {
    const { user } = ws.deserializeAttachment();
    this.broadcast({ type: "message", from: user, text: String(message) }, ws);
  }
  async webSocketClose(ws, code, reason, wasClean) { ws.close(code, reason); this.broadcast({ type: "left" }, ws); }
  async webSocketError(ws, error) { this.broadcast({ type: "left" }, ws); }
  broadcast(payload, except) {
    const text = JSON.stringify(payload);
    for (const ws of this.ctx.getWebSockets()) if (ws !== except) { try { ws.send(text); } catch {} }
  }
}
```

The client builds its URL relative to the page (any mount, any sandbox) and reconnects on every close:

```js
const url = new URL("ws?room=lobby", location.href); url.protocol = url.protocol === "https:" ? "wss:" : "ws:";
(function connect() { const ws = new WebSocket(url); ws.onmessage = (e) => render(JSON.parse(e.data)); ws.onclose = () => setTimeout(connect, 1000); })();
```

## Connections and storage

- `ctx.acceptWebSocket(ws, tags)` lets the room sleep with the socket open; never add `ws.addEventListener` to a held socket (events go to the class methods). Tags (at most 9; `ctx.getTags(ws)` also returns Yard's own `yard:connected:<ms>`) are for lookup with `ctx.getWebSockets(tag)`; attachments (`ws.serializeAttachment(value)`, `deserializeAttachment()`, a few hundred bytes) are per-connection state that survives a sleep.
- Identity arrives on the upgrade request like any request (`X-Yard-User-Id`, `X-Yard-Guest-Id`, `X-Yard-Tier-Key`, `X-Yard-Sandbox`; yard-auth.md), and the service's `access` gates the upgrade, so `"access": "users"` keeps non-buyers out of every room with no code. A socket from a page on another host arrives signed out.
- To refuse with a reason (room full, game started): accept, send the reason, then close with a code of your own from 4000 to 4999; the client reads `event.code`. Yard closes every session after 24 hours with code 1000 and reason `Session limit reached`; clients reconnect.
- Storage is per room, created on first use, no migrations: `ctx.storage.get/put/delete/list({ prefix, limit })` for structured values, `ctx.storage.sql.exec(sql, ...params)` returning a cursor (`.toArray()`, `.one()`), `ctx.storage.setAlarm(ts)`, `getAlarm()`, `deleteAlarm()` (one pending alarm per room). Keys starting with `__yard_` are Yard's (`__yard_user_alarm` shows in `list()` while an alarm is pending). A room cannot read another room's storage and the handler cannot read any of it: cross-room data goes through `env.DB`. Each sandbox has its own rooms.

## Limits, cost, lifecycle

- Storage 10 GB per room; inbound message 32 MiB; 50 ms CPU per request or message; 10 room types per service; sessions 24 h. A busy room has its own budget, which is why each conversation gets its own room.
- Monthly allowance: 1,000,000 room requests (a forwarded request, an upgrade or an alarm; 20 inbound messages count as one), 400,000 GB-seconds of compute (0.128 GB for every second a room is awake; a sleeping room costs nothing) and 2 GiB of storage; overage is billed at the allowances page's prices. A recurring alarm costs 86,400,000 / N requests a day: re-arm only while there is work (`ctx.getWebSockets().length > 0`) and use the longest interval that works.
- New code reaches rooms after the rest of the service, within seconds to minutes; a room moves to the new version when it restarts, which ends its connections (storage is kept). Keep messages between page, service and rooms compatible across one release (add fields with a fallback) and judge a deploy by something the room sends, such as a version in its first message.
- Removing an entry from `rooms` deletes every room of that type with its storage at the next deploy (`yard push` warns `room type X will be deleted with all its data`); renaming a type is a delete plus a create (rename `binding` to change the `env` name instead). A removed or renamed service leaves its rooms behind: kept 30 days on the project, deleted at once in a sandbox. Copy anything worth keeping into `env.DB` first.

Common mistakes: ports or the `ws` package (nothing listens; a socket is accepted by returning a 101 from the room's `fetch`), polling `env.DB` for presence, fanning out from the fetch handler, state in instance fields, a root-absolute WebSocket URL, writing `env.DB` on every message (buffer in storage, flush from `alarm()`), one room for everything. `yard dev` runs rooms with the same contract (local-dev.md); hosted output is in `yard service logs`.
