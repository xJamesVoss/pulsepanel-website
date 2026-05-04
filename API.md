# PulseConnect API Reference

**For developers building custom integrations.** If you want to use a Discord bot, web dashboard, CLI, or your own custom client to talk to a PulseConnect-enabled server, this is the document.

The PulsePanel iOS app is the official client and the easiest way to use PulseConnect, but the API is fully documented and not coupled to the app. Everything below is fair game for third-party integrations.

> **Heads up — this is the developer reference.** It documents the full PulseConnect API surface, including multi-user staff accounts and granular permissions. The PulsePanel iOS app uses owner-only mode and does not expose those features in its UI. If you're just running the app, you don't need this document — see the [setup guide](https://pulsepanelapp.com/setup.html) instead.

---

## Conventions

### Base URL

```
http://<your-server>:7070/api/v1
```

Default port is `7070`, configurable via `api.port` in `config.yml`. The plugin serves plain HTTP — if you're exposing the API to the internet, put it behind a reverse proxy (nginx, Caddy, Cloudflare) for HTTPS.

### Authentication

All endpoints except `POST /auth/login` and `POST /auth/register` require a JWT in the `Authorization` header:

```
Authorization: Bearer <token>
```

Tokens are obtained via login (see below). They expire after 24 hours by default (`api.token-expiry-hours` in config). When a token expires, the server returns `401 Unauthorized` and the client should re-login.

### Content type

All request bodies are JSON. All response bodies are JSON. Set `Content-Type: application/json` on requests.

### Errors

Failed requests return an HTTP error status (4xx or 5xx) with a JSON body:

```json
{ "error": "Human-readable message" }
```

Common status codes:

| Status | Meaning |
|---|---|
| `400` | Bad request (missing/malformed body) |
| `401` | Not authenticated (missing or invalid token) |
| `403` | Authenticated but lacks the required permission |
| `404` | Resource not found |
| `409` | Conflict (e.g., duplicate username, file edit conflict) |
| `429` | Too many requests (login rate limiting) |
| `500` | Server error |

### Permissions

Most endpoints require a specific permission. The Owner role bypasses all permission checks. Other staff have the permissions explicitly granted to them by the Owner.

The full permission catalog is documented at the bottom of this file.

---

## Auth

| Method | Path | Permission | Description |
|---|---|---|---|
| `POST` | `/auth/login` | none | Exchange username/password for a JWT |
| `POST` | `/auth/register` | bootstrap or Owner | Create the first user (bootstrap) or another user |
| `GET` | `/auth/me` | any authenticated | Get the caller's username, role, permissions |
| `POST` | `/auth/me/password` | any authenticated | Change own password |
| `GET` | `/auth/users` | Owner | List all staff |
| `POST` | `/auth/users` | Owner | Create a staff user |
| `DELETE` | `/auth/users/{username}` | Owner | Delete a staff user |
| `GET` | `/auth/users/{username}/permissions` | Owner | Get a user's permissions |
| `PUT` | `/auth/users/{username}/permissions` | Owner | Replace a user's permission set |
| `POST` | `/auth/users/{username}/password` | Owner | Reset another user's password |

### Bootstrap and registration

`POST /auth/register` creates user accounts. There are two distinct modes:

**Bootstrap mode** — when zero accounts exist in the database (fresh install, or after the database has been wiped), `/auth/register` is callable without authentication. The first call creates the Owner. Once any account exists, bootstrap mode closes permanently and the endpoint requires an Owner JWT.

In practice, the Owner is normally created automatically from the `admin:` block in `config.yml` on first server startup, and direct calls to `/auth/register` are only needed for the post-bootstrap case (creating additional staff users, which require Owner auth).

Request body for both modes:

```json
{
  "username": "new-user",
  "password": "their-password",
  "role": "staff"
}
```

`role` is `"owner"` or `"staff"`. Only one Owner exists per server; attempting to create a second returns `409 Conflict`.

---

### Login (worked example)

```http
POST /api/v1/auth/login
Content-Type: application/json

{
  "username": "admin",
  "password": "your-password"
}
```

Response:

```json
{
  "token": "eyJhbGciOiJIUzI1NiJ9...",
  "username": "admin",
  "role": "owner"
}
```

After 5 failed attempts in 15 minutes (configurable), the (username, IP) pair is locked out and `429 Too Many Requests` is returned. The lockout window resets on a successful login.

### `/auth/me` response shape

```json
{
  "username": "admin",
  "role": "owner",
  "permissions": ["dashboard.view", "console.view", "..."]
}
```

For Owner accounts, `permissions` is the full catalog. For other roles, it's only what the Owner has granted.

---

## Server

| Method | Path | Permission | Description |
|---|---|---|---|
| `GET` | `/server/status` | `dashboard.view` | TPS, memory, uptime, player counts |
| `POST` | `/server/reload` | `server.reload` | Reload the server config |
| `POST` | `/server/restart` | `server.restart` | Restart the server |
| `POST` | `/server/stop` | `server.stop` | Stop the server |
| `GET` | `/server/console-history` | `console.view` | Last 100 lines of server console |

### Status (worked example)

```http
GET /api/v1/server/status
Authorization: Bearer <token>
```

Response:

```json
{
  "online": true,
  "version": "Paper 1.21.4",
  "motd": "A Minecraft Server",
  "onlinePlayers": 3,
  "maxPlayers": 20,
  "tps1m": 19.98,
  "tps5m": 19.95,
  "tps15m": 19.93,
  "usedMemoryMb": 2048,
  "maxMemoryMb": 8192,
  "uptimeMs": 14_400_000
}
```

---

## Players

| Method | Path | Permission | Description |
|---|---|---|---|
| `GET` | `/players` | `players.view` | List online players |
| `GET` | `/player/{uuid}` | `players.view` | Get one player's details |
| `POST` | `/player/{uuid}/health` | `players.edit_stats` | Set health (0–20) |
| `POST` | `/player/{uuid}/hunger` | `players.edit_stats` | Set hunger (0–20) |
| `POST` | `/player/{uuid}/gamemode` | `players.edit_stats` | Set game mode |
| `POST` | `/player/{uuid}/op` | `players.op` | Grant or revoke OP |
| `GET` | `/player/{uuid}/inventory` | `players.inventory` | Inventory + armor + ender chest |
| `POST` | `/player/{uuid}/inventory/give` | `players.inventory` | Give an item |
| `POST` | `/player/{uuid}/inventory/set` | `players.inventory` | Set a specific slot |
| `DELETE` | `/player/{uuid}/inventory` | `players.inventory` | Clear inventory |

---

## Moderation

| Method | Path | Permission | Description |
|---|---|---|---|
| `POST` | `/moderation/kick` | `players.kick` | Kick a player with a reason |
| `POST` | `/moderation/ban` | `players.ban` | Ban (permanent or timed) |
| `DELETE` | `/moderation/ban/{uuid}` | `players.ban` | Unban |
| `GET` | `/moderation/bans` | `players.ban` | List active bans |
| `GET` | `/moderation/bans/{uuid}/history` | `players.ban` | Ban history for one player |
| `POST` | `/moderation/mute` | `players.mute` | Mute a player |
| `DELETE` | `/moderation/mute/{uuid}` | `players.mute` | Unmute |
| `GET` | `/moderation/mutes` | `players.mute` | List active mutes |
| `POST` | `/moderation/flag` | `players.flag` | Flag a player for staff attention |
| `DELETE` | `/moderation/flag/{uuid}` | `players.flag` | Unflag |
| `GET` | `/moderation/flagged` | `players.flag` | List flagged players |

---

## Console / Commands

| Method | Path | Permission | Description |
|---|---|---|---|
| `POST` | `/command` | `console.execute` | Execute a console command |

Body: `{ "command": "say Hello world" }`

Response: `{ "success": true, "message": "..." }`

`stop`, `restart`, and `reload` are blocked at the command level — use the dedicated `/server/*` endpoints instead.

---

## Tickets

| Method | Path | Permission | Description |
|---|---|---|---|
| `GET` | `/tickets?status={open\|closed}` | `tickets.view` | List tickets, optionally filtered |
| `GET` | `/tickets/{id}` | `tickets.view` | Ticket detail with messages |
| `POST` | `/tickets/{id}/reply` | `tickets.reply` | Add a staff reply |
| `POST` | `/tickets/{id}/close` | `tickets.manage` | Close a ticket |
| `POST` | `/tickets/{id}/reopen` | `tickets.manage` | Reopen a closed ticket |
| `POST` | `/tickets/{id}/priority` | `tickets.manage` | Set priority (low/medium/high) |
| `DELETE` | `/tickets/{id}` | `tickets.delete` | Permanently delete |

Tickets are created by players in-game via `/ticket create <message>`, not via the API. The API is staff-side only.

---

## Worlds

| Method | Path | Permission | Description |
|---|---|---|---|
| `GET` | `/worlds` | `worlds.view` | List loaded worlds |
| `GET` | `/worlds/{name}` | `worlds.view` | Detail for one world |
| `POST` | `/worlds/{name}/gamerule` | `worlds.edit` | Set a game rule |
| `POST` | `/worlds/{name}/time` | `worlds.edit` | Set time of day (0–24000) |
| `POST` | `/worlds/{name}/weather` | `worlds.edit` | Set weather (storm, thunder) |
| `POST` | `/worlds/{name}/difficulty` | `worlds.edit` | peaceful / easy / normal / hard |

---

## Analytics

| Method | Path | Permission | Description |
|---|---|---|---|
| `GET` | `/analytics/tps?hours=24` | `analytics.view` | TPS history |
| `GET` | `/analytics/sessions?hours=24` | `analytics.view` | Player session log |

---

## Files

The file editor is sandboxed to the server's `plugins/` directory. Symlinks that escape are rejected by canonical-path validation. Editable extensions: `.yml`, `.yaml`, `.json`, `.properties`, `.txt`, `.conf`. Other files are listed but not editable. 1MB file size cap.

| Method | Path | Permission | Description |
|---|---|---|---|
| `GET` | `/files/tree?path=...` | `files.view` | List a directory inside `plugins/` |
| `GET` | `/files/read?path=...` | `files.view` | Read a file (returns content + sha256 hash) |
| `POST` | `/files/write` | `files.edit` | Write a file (requires `expected_hash`) |

Writes use optimistic locking via the `expected_hash` field. If the file changed since you last read it, the server returns `409 Conflict` with the current hash so you can re-read and decide whether to overwrite.

### Write request

```json
{
  "path": "Essentials/config.yml",
  "content": "...",
  "expected_hash": "sha256-of-the-content-when-you-last-read-it"
}
```

### Conflict response (409)

```json
{
  "error": "File changed since you last read it",
  "current_hash": "sha256-of-current-content"
}
```

---

## Notifications (push)

Per-user, self-service. Used by the iOS app to register device tokens for APNs. No permission required beyond authentication.

| Method | Path | Description |
|---|---|---|
| `POST` | `/devices/register` | Register an APNs device token for the current user |
| `DELETE` | `/devices/{token}` | Unregister a device token |
| `GET` | `/notifications/preferences` | Get the user's notification preferences |
| `PUT` | `/notifications/preferences` | Update preferences |

Third-party integrations can ignore this section unless they're building their own iOS app.

---

## Audit Log

| Method | Path | Permission | Description |
|---|---|---|---|
| `GET` | `/audit?limit=100&before=<timestamp>` | Owner | Audit log entries, newest-first |

Pagination is keyset-based: pass the timestamp of the oldest row from the previous page as `before`.

---

## WebSocket

A single WebSocket connection delivers all real-time events.

```
ws://<server>:7070/api/v1/ws?token=<jwt>
```

The token is passed as a query parameter (WebSocket upgrades don't carry custom headers reliably across implementations). It's verified on connect; an invalid token closes the connection immediately with code 1008.

### Envelope

Every server-to-client message has this envelope:

```json
{
  "type": "event_type",
  "data": { ... },
  "timestamp": 1730390400000
}
```

`timestamp` is server-side milliseconds since epoch, applied at the moment the event was produced. Per-event fields go in `data`.

### Event types

| `type` | `data` fields | Description |
|---|---|---|
| `system` | `message` | Connection-level message. Sent once on connect with `"Connected to PulseConnect"` |
| `chat` | `uuid`, `name`, `message` | In-game chat message |
| `console` | `line` | New console log line. May include raw ANSI escape codes — clients can render them as colors or strip them |
| `player_join` | `uuid`, `name` | Player joined the server |
| `player_leave` | `uuid`, `name` | Player left the server |
| `flagged_player_join` | `uuid`, `name` | A player previously flagged by staff just joined. Sent in addition to `player_join` |
| `ticket_created` | `ticketId`, `playerName`, `subject` | New ticket opened |
| `ticket_message` | `ticketId`, `sender`, `senderType`, `message` | Message added to a ticket. `senderType` is `"player"` or `"staff"` |
| `ticket_update` | `ticketId`, `status?`, `priority?` | Ticket status or priority changed (close, reopen, priority change). Fields are present only if changed |
| `tps_alert` | `tps`, `message` | TPS dropped below the configured threshold |

### Example payload

```json
{
  "type": "chat",
  "data": {
    "uuid": "550e8400-e29b-41d4-a716-446655440000",
    "name": "Notch",
    "message": "hello world"
  },
  "timestamp": 1730390400000
}
```

### Client-to-server messages

Clients can send messages on the WebSocket, but the plugin currently only logs them — there are no client-initiated commands over the WebSocket. To execute a server command, use `POST /command`.

---

## Permissions Catalog

The complete list of permission keys, used in `PUT /auth/users/{username}/permissions`:

### Dashboard / Analytics
- `dashboard.view` — see the dashboard at all
- `analytics.view` — see TPS history and player session graphs

### Server control
- `server.restart` — restart the server
- `server.stop` — stop the server (**dangerous**)
- `server.reload` — reload the server config

### Console
- `console.view` — see live console logs and history
- `console.execute` — run console commands (effectively grants OP-level access)

### Players
- `players.view` — see player list and details
- `players.kick` — kick players
- `players.ban` — ban / unban players
- `players.mute` — mute / unmute players
- `players.edit_stats` — set health, hunger, game mode
- `players.op` — grant or revoke OP (**dangerous**)
- `players.inventory` — view and edit player inventories
- `players.flag` — flag players for staff attention

### Tickets
- `tickets.view` — see tickets
- `tickets.reply` — reply to tickets
- `tickets.manage` — close, reopen, set priority
- `tickets.delete` — permanently delete tickets (**dangerous**)

### Worlds
- `worlds.view` — see world settings
- `worlds.edit` — change game rules, time, weather, difficulty

### Chat
- `chat.view` — see in-game chat
- `chat.send` — send messages as staff
- `chat.broadcast` — send server-wide broadcasts

### Files
- `files.view` — list and read files in `plugins/`
- `files.edit` — write files in `plugins/` (**dangerous** — full access to plugin configs)

### Notification subscriptions
These don't unlock actions; they unlock receiving push notifications for specific events.
- `notifications.server_lag` — receive a push when TPS drops
- `notifications.server_stop` — receive a push when the server stops

### Defaults

When the Owner creates a new staff user without specifying permissions explicitly, they get a view-only set:

`dashboard.view`, `analytics.view`, `console.view`, `players.view`, `tickets.view`, `worlds.view`, `chat.view`, `files.view`, `notifications.server_lag`, `notifications.server_stop`

The Owner toggles individual write permissions on per-staff-member.

---

## Versioning

The API is versioned in the URL path (`/api/v1/...`). Breaking changes will go to `/api/v2`; `/api/v1` will continue to work alongside.

Adding new endpoints, new optional fields, or new permission keys is **not** considered breaking. Removing or renaming endpoints, removing fields, or changing field types is.

---

## Examples

### Get current player count (curl)

```bash
TOKEN=$(curl -s -X POST http://localhost:7070/api/v1/auth/login \
  -H "Content-Type: application/json" \
  -d '{"username":"admin","password":"your-password"}' \
  | jq -r .token)

curl -s http://localhost:7070/api/v1/server/status \
  -H "Authorization: Bearer $TOKEN" \
  | jq .onlinePlayers
```

### Send a broadcast from a Discord bot (Python)

```python
import requests

base = "http://your-server:7070/api/v1"

# Login once at startup
token = requests.post(f"{base}/auth/login", json={
    "username": "discord-bot",
    "password": "..."
}).json()["token"]

headers = {"Authorization": f"Bearer {token}"}

# Send a broadcast
requests.post(f"{base}/command", headers=headers, json={
    "command": "say [Discord] Server restart in 5 minutes"
})
```

### Watch chat in real time (Node)

```javascript
const WebSocket = require("ws");

const ws = new WebSocket(`ws://your-server:7070/api/v1/ws?token=${TOKEN}`);

ws.on("message", (data) => {
  const event = JSON.parse(data);
  if (event.type === "chat") {
    console.log(`<${event.data.name}> ${event.data.message}`);
  }
});
```

---

## Support

- **Bug reports / feature requests:** email [support@pulsepanelapp.com](mailto:support@pulsepanelapp.com)
- **General questions:** [pulsepanelapp.com](https://pulsepanelapp.com)
