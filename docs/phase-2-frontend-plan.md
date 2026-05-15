# Phase 2 Frontend Implementation Plan

## Objective

Build the SvelteKit static browser/mobile UI for `pi-remote` MVP.

The frontend connects to the Phase 1 WebSocket backend, authenticates with the pairing token, shows the last-3-message snapshot, streams live updates, supports text prompt submission when idle, supports abort while busy, and handles single-client rejection cleanly.

## Hard Constraints

- SvelteKit static output only.
- No frontend server runtime.
- No token persistence.
- No session data before WebSocket auth succeeds.
- Mobile-first and clean functional, not highly polished v2.
- Text-only prompt input.

## Target Frontend Structure

```text
src/web/
  package.json or integrated root scripts
  svelte.config.js
  vite.config.ts
  tsconfig.json

  src/
    app.html
    routes/
      +page.svelte

    lib/
      protocol.ts
      wsClient.ts
      stores.ts
      sessionModel.ts

      components/
        LoginCard.svelte
        ConnectionBanner.svelte
        SessionTranscript.svelte
        MessageBubble.svelte
        ToolSummary.svelte
        Composer.svelte
        BusyState.svelte
        RejectedClient.svelte
```

Build output:

```text
dist/web/
  index.html
  _app/...
```

## Step 1 — Add SvelteKit static app

Install/configure:

- `@sveltejs/kit`
- `@sveltejs/adapter-static`
- `svelte`
- `vite`
- `typescript`

Configure `adapter-static` so output lands in package-level `dist/web`.

Ensure the app works when served by Express from arbitrary host/port.

## Step 2 — Share or duplicate protocol types

Frontend should use the same protocol shapes as backend where practical.

Options:

- Shared generated/compiled TS module from `src/protocol/messages.ts`.
- Frontend-local copy with tests/manual sync.

Recommendation: keep protocol types in a shared source module importable by both backend and frontend build.

## Step 3 — WebSocket client

Create `src/web/src/lib/wsClient.ts`.

Responsibilities:

- Connect to `ws://<current-host>/` or `/ws` depending backend route.
- Expose connection states:
  - disconnected
  - connecting
  - awaiting_auth
  - authenticated
  - rejected
  - error
- Send auth token manually entered by user.
- Keep token only in memory.
- Parse incoming JSON messages safely.
- Route events into stores.
- Support in-page reconnect using memory token if connection drops.

No localStorage/sessionStorage/cookies.

## Step 4 — App state stores

Create `stores.ts` / `sessionModel.ts`.

State should include:

- auth status
- connection status
- rejection reason
- session metadata
- messages
- tool summaries
- busy/idle state
- current error/toast message
- prompt draft

Message update logic must support streaming updates by id.

## Step 5 — Login screen

Create `LoginCard.svelte`.

Behavior:

- Shows project name and short explanation.
- Token input.
- Connect/authenticate button.
- Error display for wrong token.
- Never stores token persistently.

Copy should make clear this controls a live pi session.

## Step 6 — Rejected-client screen

Create `RejectedClient.svelte`.

If backend sends `client_rejected` with `client_already_connected`:

- Show clear message: another remote browser is already connected.
- Offer retry button.
- Do not show session data.

## Step 7 — Connection banner

Create `ConnectionBanner.svelte`.

Show:

- connecting/authenticated/disconnected/reconnecting
- session cwd/name if available
- model provider/id if available
- busy/idle state

Keep compact for mobile.

## Step 8 — Transcript rendering

Create:

- `SessionTranscript.svelte`
- `MessageBubble.svelte`
- `ToolSummary.svelte`

Requirements:

- Render last-3 snapshot messages on auth.
- Append/update live events.
- User and assistant messages visually distinct.
- Assistant streaming should update smoothly without full-page flicker.
- Tool summaries collapsed by default.
- Tool summary includes name, state, and short preview.
- Auto-scroll near bottom when new content arrives, but avoid fighting user scroll if they scroll up.

## Step 9 — Composer

Create `Composer.svelte`.

Requirements:

- Sticky bottom composer on mobile.
- Textarea for text-only prompts.
- Send button enabled only when authenticated and idle.
- `Enter` sends, `Shift+Enter` inserts newline.
- If busy, prompt controls disabled with small explanation.
- Abort button shown/enabled when busy.
- Prompt text is sent as-is, including slash-command-looking text.

Client messages:

```ts
{ type: "prompt", text }
{ type: "abort" }
```

## Step 10 — Responsive styling

MVP styling goals:

- Mobile-first single-column layout.
- Readable transcript.
- Large tap targets.
- Sticky composer.
- Safe-area padding for mobile browsers.
- Light/dark aware if easy via CSS media query.
- No advanced animation required.

Avoid heavy UI libraries for MVP unless strongly justified.

## Step 11 — Build/package integration

Root package scripts should support:

```bash
npm run build:extension
npm run build:web
npm run build
```

`npm run build:web` should output to `dist/web`.

Published package should include:

- `dist/extension/**`
- `dist/web/**`
- `README.md`
- docs as desired

Installed package should not require the user to run Vite/SvelteKit.

## Step 12 — Frontend verification checklist

Manual checks:

1. Load URL before auth; no session content visible.
2. Wrong token shows error.
3. Correct token shows app shell and snapshot.
4. Snapshot renders last 3 user/assistant messages.
5. Assistant text streams live.
6. Tool summaries appear collapsed.
7. Prompt button disabled while busy.
8. Prompt submits while idle.
9. Abort appears/works while busy.
10. Reload forgets token and returns to login.
11. Disconnect shows reconnecting/disconnected state.
12. Second client rejection displays clear screen.
13. UI works on phone-size viewport.

## Phase 2 Deliverable

A prebuilt SvelteKit static UI in `dist/web` that provides the complete MVP browser/mobile experience over the Phase 1 backend protocol.
