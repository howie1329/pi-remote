# pi-remote MVP

## MVP Summary

`pi-remote` MVP exposes the active pi TUI session to exactly one authenticated browser/mobile client over LAN or Tailscale.

The user starts it from pi:

```text
/remote
```

The extension starts an embedded HTTP/WebSocket server, prints URLs and a fresh pairing token, serves a SvelteKit static UI, and lets the connected browser view the live session, submit text prompts when pi is idle, and abort active work.

## Locked MVP Scope

### Connection model

- Single authenticated remote client only.
- Reject a second authenticated client while one is connected.
- No multi-viewer support in MVP.
- No controller request/release flow in MVP.
- Browser disconnect frees the single-client slot.
- Server remains running after browser disconnect.

### Auth model

- Static web UI can load without auth.
- WebSocket requires pairing-token auth before any session data or controls are available.
- Token is generated fresh on each server start.
- Token is stored only in memory.
- Token is invalidated on `/remote stop`, `/remote restart`, pi shutdown, reload, or session replacement.
- Browser does not persist the token in localStorage, sessionStorage, cookies, or disk.

### Network model

- Default bind address is `0.0.0.0`.
- MVP uses HTTP and plain WebSocket.
- Intended use is LAN or Tailscale, not raw public internet.
- Print localhost, LAN, and Tailscale URLs when available.

### Remote controls

MVP controls are intentionally minimal:

- Submit text prompt when pi is idle.
- Abort active work when pi is busy.

Not included in MVP:

- Steering messages.
- Follow-up queueing.
- Image attachments.
- Multi-controller logic.

### Prompt behavior

- Text-only prompts.
- Remote prompt is sent as-is, including slash-command-looking text.
- Local TUI input remains usable while remote is connected.
- Remote can submit even if local TUI editor contains unsent draft text.
- Busy state disables prompt submission or returns a structured busy error.

### Snapshot behavior

On authenticated connect, send:

- Current session metadata.
- Active branch only.
- Last 3 user/assistant conversational messages.
- Related tool activity nested under associated assistant messages where practical.

Do not expose:

- Full session tree.
- Branch navigation.
- Historical browsing beyond last 3 messages.

### Live streaming behavior

After connect, browser mirrors all live session events regardless of whether they originated locally or remotely:

- User messages.
- Assistant streaming updates.
- Assistant finalization.
- Tool start/update/end summaries.
- Agent busy/idle state.

### Tool display

- Show collapsed tool summaries.
- Include tool name, running/success/error state, and short preview when available.
- Do not inline large tool output in MVP.

### Frontend

- SvelteKit static frontend.
- Clean functional mobile-first UI.
- Prebuilt assets included in package.
- Runtime serves package-relative `dist/web`.

### Package/runtime

- Compiled JS pi extension entrypoint.
- `pi.extensions` points to compiled output.
- Extension attaches to active pi TUI session.
- Extension must not create a separate SDK session.

## Out of Scope for MVP

- HTTPS/WSS.
- Multi-viewer support.
- Controller election.
- Force takeover.
- Follow-up and steering controls.
- Image uploads.
- Full session tree UI.
- Remote branch navigation.
- Rich design polish/animations.
- Token persistence.
- Public internet hardening.

## Success Criteria

MVP is complete when:

1. `/remote` starts an embedded server from an active pi session.
2. Server prints usable localhost, LAN, and Tailscale URLs when available.
3. Server prints a fresh pairing token.
4. Browser can load the SvelteKit UI.
5. Browser must authenticate over WebSocket before receiving session data.
6. Authenticated browser receives last 3 active-branch user/assistant messages.
7. Browser receives live assistant streaming and tool summary updates.
8. Browser can submit a text prompt when pi is idle.
9. Browser can abort while pi is busy.
10. A second authenticated browser is rejected.
11. `/remote`, `/remote status`, `/remote stop`, and `/remote restart` work.
12. Server shuts down cleanly on pi shutdown/reload/session replacement.
