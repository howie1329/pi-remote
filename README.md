# pi-remote

`pi-remote` is a planned installable [pi](https://pi.dev) extension package that exposes an **active pi TUI session** to one authenticated browser/mobile client over LAN or Tailscale.

From inside a running pi session:

```text
/remote
```

The extension starts an embedded HTTP/WebSocket server, prints local/LAN/Tailscale URLs plus a pairing token, serves a prebuilt SvelteKit static UI, and lets the authenticated browser mirror the live session, submit text prompts when pi is idle, and abort active work.

> Status: MVP planning. Implementation has not started yet.

## MVP Scope

The MVP is intentionally single-user and focused:

- One authenticated remote browser/mobile client at a time.
- Reject second authenticated clients.
- Static web UI can load openly, but WebSocket auth gates all session data and controls.
- Fresh in-memory pairing token per server start.
- Default bind address: `0.0.0.0` for LAN/Tailscale use.
- HTTP/WebSocket for MVP.
- Snapshot-on-connect: active branch only, last 3 user/assistant messages.
- Live assistant streaming after connect.
- Collapsed tool summaries.
- Remote controls: text prompt when idle and abort while busy.
- SvelteKit static frontend, prebuilt and served from package-relative `dist/web`.
- Compiled JS pi extension entrypoint.

See [`docs/mvp.md`](./docs/mvp.md) for the full locked MVP definition.

## Non-goals

- Not a standalone pi SDK session host.
- Not a multi-viewer collaboration server in MVP.
- No controller election/request/release flow in MVP.
- No steering/follow-up controls in MVP.
- No image attachments in MVP.
- No full session tree or branch navigation in MVP.
- No token persistence in the browser.
- No HTTPS/WSS in MVP.
- Not intended for direct public-internet exposure.

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

Open one of the URLs, enter the pairing token, and use the browser/mobile UI as a remote control surface for the active pi session.

## Planned commands

```text
/remote                  Start server, or show status if already running
/remote status           Show URLs, token, bind address, and client state
/remote stop             Stop the embedded server
/remote restart          Restart server with a fresh token
```

Potential later flags:

```text
/remote --port 49231
/remote --host 0.0.0.0
```

## Security model

`pi-remote` intentionally exposes a live local agent session over the network. Treat it as powerful remote-control software.

MVP safeguards:

- Pairing-token auth required before any session data is sent.
- Token is generated fresh on every server start.
- Token is stored only in memory.
- Browser does not persist token.
- Only one authenticated client is allowed.
- Second authenticated clients are rejected.
- Server shuts down on pi session shutdown/reload/session replacement.

Use over trusted LAN or Tailscale. Do **not** expose directly to the public internet.

## Architecture

```text
pi-remote/
  README.md

  docs/
    mvp.md
    phase-1-backend-plan.md
    phase-2-frontend-plan.md

  src/
    extension/
      index.ts                  # pi extension entrypoint
      commands.ts               # /remote command parsing
      state.ts                  # extension runtime state

    server/
      createRemoteServer.ts     # Express + WebSocket server
      lifecycle.ts              # start/stop/restart/status
      auth.ts                   # pairing-token auth
      clients.ts                # single authenticated client policy
      urls.ts                   # printed URL construction
      tailscale.ts              # Tailscale IP discovery

    protocol/
      messages.ts               # typed client/server messages
      snapshots.ts              # snapshot protocol types
      events.ts                 # live event protocol types

    pi/
      bridge.ts                 # active pi runtime bridge
      controls.ts               # prompt/abort control mapping
      sessionSnapshot.ts        # last-3 active branch snapshot
      normalizeMessage.ts       # safe message normalization

    web/
      ...                       # SvelteKit static frontend source

  dist/
    extension/
      index.js                  # compiled extension entrypoint
    web/
      ...                       # prebuilt SvelteKit static assets
```

## pi integration

The package should expose the compiled extension through the pi package manifest:

```json
{
  "keywords": ["pi-package", "pi-extension"],
  "pi": {
    "extensions": ["./dist/extension/index.js"]
  }
}
```

The extension must attach to the active pi runtime:

```ts
import type { ExtensionAPI } from "@earendil-works/pi-coding-agent";

export default function piRemote(pi: ExtensionAPI) {
  pi.registerCommand("remote", {
    description: "Start remote browser/mobile access for this pi session",
    handler: async (args, ctx) => {
      // start/status/stop/restart server
    },
  });

  pi.on("message_update", async (event, ctx) => {
    // broadcast normalized live updates
  });

  pi.on("session_shutdown", async () => {
    // stop server and clean up
  });
}
```

It must **not** call `createAgentSession()` for the remote UI.

## Implementation plans

Implementation is split into two phases:

1. [`docs/phase-1-backend-plan.md`](./docs/phase-1-backend-plan.md) — pi extension backend, embedded server, auth, snapshot, streaming, prompt/abort controls.
2. [`docs/phase-2-frontend-plan.md`](./docs/phase-2-frontend-plan.md) — SvelteKit static browser/mobile UI.

## License

TBD.
