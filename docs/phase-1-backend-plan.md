# Phase 1 Backend Implementation Plan

## Objective

Build the pi extension backend for `pi-remote`.

Phase 1 delivers the installable package runtime, `/remote` command, embedded HTTP/WebSocket server, token auth, single-client connection policy, active-session snapshot, live pi event streaming, prompt/abort controls, and LAN/Tailscale URL discovery.

The backend should work with a placeholder static page until Phase 2 provides the final SvelteKit UI.

## Hard Constraints

- Must attach to the active pi TUI extension runtime.
- Must not create a new pi SDK session.
- Must expose no session data before WebSocket auth succeeds.
- Must allow only one authenticated remote client in MVP.
- Must stop cleanly on `session_shutdown`.
- Must avoid stale contexts after reload/session replacement.

## Target Package Structure

```text
src/
  extension/
    index.ts
    commands.ts
    state.ts

  server/
    createRemoteServer.ts
    lifecycle.ts
    auth.ts
    clients.ts
    urls.ts
    tailscale.ts

  protocol/
    messages.ts
    snapshots.ts
    events.ts

  pi/
    bridge.ts
    controls.ts
    sessionSnapshot.ts
    normalizeMessage.ts

  web-placeholder/
    index.html
```

Build output:

```text
dist/
  extension/
    index.js
  web/
    index.html      # placeholder in phase 1; SvelteKit output in phase 2
```

## Step 1 — Package skeleton

Create:

- `package.json`
- `tsconfig.json`
- `.gitignore`
- source directories
- build scripts

Package manifest should eventually expose:

```json
{
  "type": "module",
  "keywords": ["pi-package", "pi-extension"],
  "pi": {
    "extensions": ["./dist/extension/index.js"]
  }
}
```

Initial runtime dependencies:

- `@earendil-works/pi-coding-agent`
- `express`
- `ws`
- `nanoid`

Dev dependencies:

- `typescript`
- `@types/express`
- `@types/ws`
- `@types/node`

## Step 2 — Define protocol types

Create `src/protocol/messages.ts`.

Client messages:

```ts
export type ClientToServerMessage =
  | { type: "auth"; token: string }
  | { type: "prompt"; text: string }
  | { type: "abort" }
  | { type: "ping" };
```

Server messages:

```ts
export type ServerToClientMessage =
  | { type: "auth_ok"; clientId: string }
  | { type: "auth_error"; reason: string }
  | { type: "client_rejected"; reason: "client_already_connected" }
  | { type: "snapshot"; snapshot: SessionSnapshot }
  | { type: "message_start"; message: RemoteMessage }
  | { type: "message_update"; message: RemoteMessage }
  | { type: "message_end"; message: RemoteMessage }
  | { type: "tool_start"; tool: RemoteToolEvent }
  | { type: "tool_update"; tool: RemoteToolEvent }
  | { type: "tool_end"; tool: RemoteToolEvent }
  | { type: "agent_state"; state: "idle" | "busy" }
  | { type: "control_error"; reason: string }
  | { type: "pong" };
```

Keep protocol JSON-only.

## Step 3 — Server lifecycle

Create `src/server/lifecycle.ts` with a singleton-like server state owned by the extension instance.

Required operations:

- `startRemoteServer(options)`
- `stopRemoteServer(reason)`
- `restartRemoteServer(options)`
- `getRemoteServerStatus()`

Server status should include:

- running/stopped
- host
- port
- token
- URLs
- authenticated client count
- current client id if connected

`/remote` behavior:

- If stopped: start server and print status.
- If running: print status.
- `/remote status`: print status.
- `/remote stop`: stop server.
- `/remote restart`: stop, start with new token.

## Step 4 — Express + WebSocket server

Create `src/server/createRemoteServer.ts`.

Responsibilities:

- Bind to host/port. Default host: `0.0.0.0`; default port: `0`.
- Serve static files from package-relative `dist/web`.
- Create WebSocket server on same HTTP server.
- Parse JSON client messages safely.
- Route authenticated control messages.
- Close HTTP server cleanly.

Static HTTP must not expose session data.

## Step 5 — Auth and single-client policy

Create:

- `src/server/auth.ts`
- `src/server/clients.ts`

Rules:

1. Client connects unauthenticated.
2. Server waits for `{ type: "auth", token }`.
3. If token is wrong, send `auth_error` and close or remain locked.
4. If token is right but an authenticated client already exists, send `client_rejected` and close.
5. If token is right and no client exists, assign `clientId`, mark authenticated, send `auth_ok`, then send snapshot.
6. On disconnect, clear authenticated client slot.

## Step 6 — URL discovery

Create:

- `src/server/urls.ts`
- `src/server/tailscale.ts`

Print URLs:

- localhost
- LAN IPv4 addresses
- Tailscale IPv4 address when available

Tailscale discovery order:

1. Try `tailscale ip -4`.
2. Fallback to network interfaces in `100.64.0.0/10`.

LAN discovery:

- Use `os.networkInterfaces()`.
- Include non-internal IPv4 addresses.
- Exclude Tailscale address from normal LAN list if separately labeled.

## Step 7 — Pi bridge state

Create `src/pi/bridge.ts`.

Backend needs access to current pi state for snapshots and controls.

Maintain a small bridge object that stores:

- `pi: ExtensionAPI`
- latest usable `ctx`
- `isBusy`
- callbacks for broadcasting events

Update bridge context from command handlers and relevant pi event handlers.

Be careful:

- Do not use stale contexts after `session_shutdown`.
- Clear bridge state during shutdown.

## Step 8 — Snapshot serialization

Create `src/pi/sessionSnapshot.ts`.

Snapshot must include:

- session file if available
- session name if available
- cwd
- model provider/id if available
- idle/busy state
- active leaf id
- last 3 active-branch user/assistant messages

Message normalization should:

- include only user/assistant conversational messages in the count
- nest related tool summaries if practical
- avoid huge raw outputs
- tolerate unknown internal message shapes

## Step 9 — Live pi event streaming

Subscribe in extension entrypoint:

- `message_start`
- `message_update`
- `message_end`
- `tool_execution_start`
- `tool_execution_update`
- `tool_execution_end`
- `agent_start`
- `agent_end`
- `session_shutdown`

Broadcast only to authenticated client.

Normalize each event before sending. Never send raw internal objects if they include excessive or unstable data.

## Step 10 — Prompt and abort controls

Create `src/pi/controls.ts`.

Prompt behavior:

- Accept only from authenticated client.
- If pi is busy, return `control_error`.
- If idle, call `pi.sendUserMessage(text)`.
- Text is sent as-is.

Abort behavior:

- Accept only from authenticated client.
- If pi is busy and current context supports abort, call `ctx.abort()`.
- If idle, return no-op or `control_error`.

## Step 11 — TUI status and notifications

When server starts:

- notify user with URLs and token
- set footer/status indicator like `remote: on :49231`

When server stops:

- clear status indicator

When second client is rejected:

- optional TUI notification

## Step 12 — Shutdown/reload cleanup

On `session_shutdown`:

- stop server
- close websockets
- close HTTP server
- clear token
- clear client state
- clear bridge context
- clear TUI status

## Step 13 — Backend verification checklist

Manual checks:

1. Start pi with local extension.
2. Run `/remote`.
3. Confirm URLs and token print.
4. Open browser; static page loads.
5. Wrong token is rejected.
6. Correct token authenticates.
7. Snapshot contains last 3 user/assistant messages only.
8. Prompt while idle submits to pi.
9. Prompt while busy is rejected.
10. Abort while busy aborts pi.
11. Second browser is rejected after first authenticates.
12. Disconnect first browser; reconnect works with same token.
13. `/remote` while running prints status.
14. `/remote stop` stops server.
15. `/remote restart` creates new token.
16. `/reload` or pi exit stops server cleanly.

## Phase 1 Deliverable

A backend-complete pi extension package that can serve a placeholder frontend, authenticate one remote browser, stream session state/events, accept prompt/abort controls, and manage lifecycle safely.
