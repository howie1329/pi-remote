# pi-remote Implementation Plan

## Goal

Build a separate installable pi package named `pi-remote`.

From an active pi TUI session:

```text
/remote
```

starts an embedded local Express + WebSocket server that mirrors and controls the same live pi session from a browser/mobile UI over LAN/Tailscale.

This is **not** a standalone SDK session host. It attaches to the currently running pi session via the extension API.

## Package shape

```text
pi-remote/
  package.json
  tsconfig.json
  README.md

  docs/
    implementation-plan.md

  src/
    index.ts                    # pi extension entrypoint

    server/
      createRemoteServer.ts     # Express + WS bootstrap
      auth.ts                   # pairing-token auth
      tailscale.ts              # Tailscale IP discovery
      lifecycle.ts              # start/stop/restart server
      clients.ts                # viewer/controller registry

    protocol/
      messages.ts               # typed client/server protocol
      snapshots.ts              # session snapshot serialization
      events.ts                 # live pi event normalization

    pi/
      bridge.ts                 # ExtensionAPI/ctx -> remote protocol bridge
      controls.ts               # prompt/steer/follow-up/abort handlers
      sessionSnapshot.ts        # snapshot current branch/messages

    web/
      index.html
      src/
        main.tsx
        App.tsx
        wsClient.ts
        components/
          SessionView.tsx
          Composer.tsx
          ConnectionStatus.tsx
          ControllerBadge.tsx
```

## `package.json`

```json
{
  "name": "pi-remote",
  "version": "0.1.0",
  "description": "Remote browser/mobile UI for an active pi TUI session",
  "type": "module",
  "keywords": ["pi-package", "pi-extension"],
  "main": "./src/index.ts",
  "pi": {
    "extensions": ["./src/index.ts"]
  },
  "dependencies": {
    "@earendil-works/pi-coding-agent": "^0.0.0",
    "express": "^5.0.0",
    "ws": "^8.18.0",
    "nanoid": "^5.0.0"
  },
  "devDependencies": {
    "@types/express": "^5.0.0",
    "@types/ws": "^8.5.0",
    "typescript": "^5.0.0",
    "vite": "^7.0.0",
    "react": "^19.0.0",
    "react-dom": "^19.0.0"
  }
}
```

Version should be pinned to the actual installed pi package version once implemented.

## Extension command

Register:

```ts
pi.registerCommand("remote", {
  description: "Start remote browser/mobile access for this pi session",
  handler: async (args, ctx) => {
    // start server if not running
    // print LAN + Tailscale URLs
    // show pairing token
    // set footer/status indicator
  },
});
```

Optional command args:

```text
/remote
/remote stop
/remote status
/remote restart
/remote --port 0
/remote --host 0.0.0.0
```

Default behavior:

- Bind to `0.0.0.0`
- Pick an available port
- Generate a short-lived pairing token
- Print URLs:
  - `http://localhost:<port>`
  - `http://<LAN-IP>:<port>`
  - `http://<TAILSCALE-IP>:<port>` if available

## Core pi bridge

The extension should subscribe to pi events and broadcast normalized events:

```ts
pi.on("session_start", ...)
pi.on("message_start", ...)
pi.on("message_update", ...)
pi.on("message_end", ...)
pi.on("tool_execution_start", ...)
pi.on("tool_execution_update", ...)
pi.on("tool_execution_end", ...)
pi.on("agent_start", ...)
pi.on("agent_end", ...)
pi.on("session_shutdown", ...)
```

Remote clients receive:

```ts
type RemoteEvent =
  | { type: "snapshot"; snapshot: SessionSnapshot }
  | { type: "message_start"; message: RemoteMessage }
  | { type: "message_update"; message: RemoteMessage }
  | { type: "message_end"; message: RemoteMessage }
  | { type: "tool_start"; tool: RemoteToolEvent }
  | { type: "tool_update"; tool: RemoteToolEvent }
  | { type: "tool_end"; tool: RemoteToolEvent }
  | { type: "agent_start" }
  | { type: "agent_end" }
  | { type: "controller_changed"; controllerClientId?: string };
```

## Snapshot-on-connect

When a browser connects and authenticates:

1. Server validates pairing token.
2. Server registers client as viewer.
3. Server immediately sends:

```ts
{
  type: "snapshot",
  snapshot: {
    sessionFile,
    sessionName,
    cwd,
    model,
    isIdle,
    hasPendingMessages,
    leafId,
    entries,
    branch
  }
}
```

Use:

```ts
ctx.sessionManager.getEntries()
ctx.sessionManager.getBranch()
ctx.sessionManager.getLeafId()
ctx.sessionManager.getSessionFile()
ctx.model
ctx.isIdle()
ctx.hasPendingMessages()
```

## Auth model

### Pairing token

On `/remote` start:

```ts
const token = nanoid(8);
```

Print:

```text
Remote pi session started

Local:     http://localhost:49231
LAN:       http://192.168.1.45:49231
Tailscale: http://100.x.y.z:49231

Pairing token: 8391-2044
```

Browser flow:

1. User opens URL.
2. UI asks for pairing token.
3. Client sends:

```ts
{
  type: "auth",
  token: "8391-2044"
}
```

4. Server replies:

```ts
{
  type: "auth_ok",
  clientId,
  role: "viewer" | "controller"
}
```

Token should be stored only in memory. A later version can add token rotation.

## Multi-viewer / one-controller policy

All authenticated clients can view.

Only one client can control at a time.

No force takeover.

Protocol:

```ts
type ClientToServer =
  | { type: "auth"; token: string }
  | { type: "request_control" }
  | { type: "release_control" }
  | { type: "prompt"; text: string }
  | { type: "steer"; text: string }
  | { type: "follow_up"; text: string }
  | { type: "abort" };
```

Rules:

- First client may request controller.
- If controller exists, other clients get:

```ts
{
  type: "control_denied",
  reason: "controller_exists"
}
```

- No force takeover.
- If controller disconnects, controller role is released.
- Server broadcasts controller changes.

## Remote controls

Map browser actions to pi extension APIs.

### Prompt

If idle:

```ts
pi.sendUserMessage(text);
```

### Steering

If streaming:

```ts
pi.sendUserMessage(text, { deliverAs: "steer" });
```

If idle, either reject or treat as prompt. Prefer reject with clear UI feedback.

### Follow-up

```ts
pi.sendUserMessage(text, { deliverAs: "followUp" });
```

### Abort

Use:

```ts
ctx.abort();
```

Available through extension context.

Because command context and event context differ, the bridge should retain the most recent valid `ExtensionContext` from event handlers/command startup.

## Tailscale IP discovery

Implement best-effort detection.

Order:

1. Run:

```bash
tailscale ip -4
```

2. Fallback: inspect network interfaces for `100.64.0.0/10`.

Node implementation:

```ts
import os from "node:os";

function findTailscaleIpFromInterfaces(): string | undefined {
  for (const iface of Object.values(os.networkInterfaces())) {
    for (const addr of iface ?? []) {
      if (addr.family === "IPv4" && isTailscale100Range(addr.address)) {
        return addr.address;
      }
    }
  }
}
```

LAN discovery should similarly scan non-internal IPv4 addresses.

## Web UI

Minimal first version:

```text
┌────────────────────────────────────┐
│ pi remote                          │
│ Connected · Viewer/Controller      │
├────────────────────────────────────┤
│ Session transcript                 │
│ - user messages                    │
│ - assistant streaming              │
│ - tool calls/results               │
├────────────────────────────────────┤
│ [Request control]                  │
│                                    │
│ textarea                           │
│ [Prompt] [Steer] [Follow-up]       │
│ [Abort]                            │
└────────────────────────────────────┘
```

Mobile-friendly requirements:

- Single-column layout
- Sticky composer
- Large tap targets
- Connection status
- Controller badge
- Streaming message updates without full rerender flicker

## Important architectural constraint

Do **not** create a new pi SDK session.

The server must be attached to the active TUI extension runtime.

Correct:

```ts
export default function(pi: ExtensionAPI) {
  pi.registerCommand("remote", ...)
  pi.on("message_update", ...)
}
```

Incorrect:

```ts
createAgentSession(...)
```

That would host a separate session and violate the goal.

## Suggested implementation phases

### Phase 1 — Package skeleton

- Create npm package
- Add pi manifest
- Add `/remote` command
- Start/stop Express server
- Serve placeholder HTML

### Phase 2 — WebSocket auth

- Add pairing token
- Add auth handshake
- Reject unauthenticated messages
- Track connected clients

### Phase 3 — Snapshot protocol

- Serialize current session branch
- Send snapshot on connect
- Render transcript in browser

### Phase 4 — Live event streaming

- Broadcast pi message/tool/agent lifecycle events
- Support streaming assistant updates
- Add reconnect behavior

### Phase 5 — Control protocol

- Add one-controller registry
- Implement request/release control
- Wire prompt / steer / follow-up / abort

### Phase 6 — URL discovery and UX

- LAN IP discovery
- Tailscale IP discovery
- Printed URLs
- TUI status indicator
- `/remote status`
- `/remote stop`

### Phase 7 — Polish

- Mobile UI
- Better transcript rendering
- Token rotation
- Optional QR code
- Tests for protocol/client registry/auth

## Key risks

1. **Abort context availability**
   - `ctx.abort()` is context-bound; bridge should retain a live context safely.

2. **Message shape stability**
   - Pi session entries may have internal shapes; isolate normalization in `sessionSnapshot.ts`.

3. **Reload/shutdown cleanup**
   - Must stop server on `session_shutdown`.
   - Must avoid stale contexts after `/reload`, `/new`, `/resume`, `/fork`.

4. **Security**
   - Bind to LAN intentionally.
   - Token auth is mandatory.
   - No unauthenticated static API that leaks session contents.
   - Consider showing a clear warning when exposing over LAN/Tailscale.

5. **No force takeover**
   - Enforce server-side, not only in UI.
