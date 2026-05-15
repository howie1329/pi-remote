# pi-remote

`pi-remote` is a planned installable [pi](https://pi.dev) extension package that exposes an **active pi TUI session** to a browser or mobile UI over LAN or Tailscale.

From inside a running pi session, the user runs:

```text
/remote
```

The extension starts an embedded Express/WebSocket server, prints local/LAN/Tailscale URLs, and lets authenticated browser clients view the same live session. One connected client may act as the controller and send prompts, steering messages, follow-ups, or abort requests.

> Status: design scaffold. See [`docs/implementation-plan.md`](./docs/implementation-plan.md) for the current build plan.

## Goals

- Attach to the **current active pi TUI session**.
- Start and stop from a pi slash command: `/remote`.
- Serve a browser/mobile UI over LAN or Tailscale.
- Use pairing-token authentication.
- Support multiple viewers.
- Allow exactly one controller at a time.
- Do not allow force takeover of the controller role.
- Send a session snapshot immediately on connect.
- Stream live pi session events to connected clients.
- Support remote prompt, steer, follow-up, and abort controls.
- Discover Tailscale IPs for printed URLs.
- Ship as a reusable pi extension package, not as a standalone pi SDK host.

## Non-goals

- This is **not** a separate web-hosted agent runtime.
- This should **not** call `createAgentSession()` or start a new pi SDK session.
- This should not expose session contents without authentication.
- This should not allow controller takeover while another client is controlling the session.

## Intended usage

After implementation and installation:

```bash
pi install npm:pi-remote
```

Then inside an interactive pi session:

```text
/remote
```

Expected output:

```text
Remote pi session started

Local:     http://localhost:49231
LAN:       http://192.168.1.45:49231
Tailscale: http://100.x.y.z:49231

Pairing token: 8391-2044
```

Open one of the URLs in a browser, enter the pairing token, and connect as a viewer. A viewer can request control if no other controller is active.

## Planned commands

```text
/remote                  Start the remote server if it is not already running
/remote status           Show server status, URLs, viewers, and controller state
/remote stop             Stop the embedded server
/remote restart          Restart the embedded server
/remote --port 0         Start on an automatically assigned port
/remote --host 0.0.0.0   Bind to a specific host
```

## Security model

`pi-remote` intentionally exposes an active local development session over the network. Treat it as powerful remote-control software.

Planned safeguards:

- In-memory pairing token required before any session data is sent.
- No unauthenticated WebSocket control messages.
- No unauthenticated HTTP API exposing session contents.
- Multiple viewers allowed, but only one controller.
- No force takeover.
- Controller role is released on disconnect.
- Server stops on pi session shutdown/reload.

Recommended default behavior:

- Bind explicitly and visibly.
- Print a clear warning when exposing over LAN/Tailscale.
- Rotate token on restart.
- Keep token out of persistent storage.

## Architecture

```text
pi-remote/
  docs/
    implementation-plan.md

  src/
    index.ts                    # pi extension entrypoint

    server/
      createRemoteServer.ts     # Express + WebSocket server
      auth.ts                   # pairing-token authentication
      clients.ts                # viewer/controller registry
      lifecycle.ts              # start/stop/restart server state
      tailscale.ts              # LAN/Tailscale IP discovery

    protocol/
      messages.ts               # typed client/server messages
      snapshots.ts              # session snapshot protocol
      events.ts                 # normalized pi live events

    pi/
      bridge.ts                 # pi ExtensionAPI/context bridge
      controls.ts               # prompt/steer/follow-up/abort handlers
      sessionSnapshot.ts        # active session serialization

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

## pi integration

The extension should be registered through the pi package manifest:

```json
{
  "keywords": ["pi-package", "pi-extension"],
  "pi": {
    "extensions": ["./src/index.ts"]
  }
}
```

The extension entrypoint should look conceptually like this:

```ts
import type { ExtensionAPI } from "@earendil-works/pi-coding-agent";

export default function piRemote(pi: ExtensionAPI) {
  pi.registerCommand("remote", {
    description: "Start remote browser/mobile access for this pi session",
    handler: async (args, ctx) => {
      // Start, stop, restart, or report status.
    },
  });

  pi.on("message_update", async (event, ctx) => {
    // Broadcast normalized live session updates.
  });

  pi.on("session_shutdown", async () => {
    // Stop server and clean up clients.
  });
}
```

## Protocol overview

Browser clients connect over WebSocket and authenticate first:

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

Server messages include authentication results, snapshots, live session events, and controller changes:

```ts
type ServerToClient =
  | { type: "auth_ok"; clientId: string; role: "viewer" | "controller" }
  | { type: "auth_error"; reason: string }
  | { type: "snapshot"; snapshot: SessionSnapshot }
  | { type: "message_start"; message: RemoteMessage }
  | { type: "message_update"; message: RemoteMessage }
  | { type: "message_end"; message: RemoteMessage }
  | { type: "tool_start"; tool: RemoteToolEvent }
  | { type: "tool_update"; tool: RemoteToolEvent }
  | { type: "tool_end"; tool: RemoteToolEvent }
  | { type: "agent_start" }
  | { type: "agent_end" }
  | { type: "controller_changed"; controllerClientId?: string }
  | { type: "control_denied"; reason: "controller_exists" };
```

## Remote controls

Planned mappings:

- Prompt while idle: `pi.sendUserMessage(text)`
- Steering while active: `pi.sendUserMessage(text, { deliverAs: "steer" })`
- Follow-up: `pi.sendUserMessage(text, { deliverAs: "followUp" })`
- Abort: `ctx.abort()`

Control messages must be accepted only from the current controller client.

## Development plan

1. Package skeleton and pi manifest.
2. `/remote` command with server lifecycle management.
3. WebSocket auth and client registry.
4. Snapshot-on-connect.
5. Live pi event streaming.
6. One-controller control protocol.
7. LAN/Tailscale URL discovery.
8. Browser/mobile UI polish.
9. Tests for protocol, auth, client registry, and lifecycle behavior.

## Documentation

- [`docs/implementation-plan.md`](./docs/implementation-plan.md) — detailed implementation plan.

## License

TBD.
