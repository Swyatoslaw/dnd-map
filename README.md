# P2P Tabletop Map (VTT)

A **serverless / peer-to-peer** Virtual Tabletop (VTT) for running tabletop RPG
sessions (D&D-style). The whole app is a **single `index.html`** file — no
backend of our own. Players connect directly to each other over WebRTC, using
the Nostr network only for signaling (peer discovery/handshake).

> UI language is **Russian**. Code comments are mixed RU/EN.

---

## Stack

- **HTML5 / CSS3 / Vanilla JavaScript** (ES6 Modules, no build step).
- **[Trystero](https://github.com/dmotz/trystero)** `@0.22.0` — P2P sync over
  WebRTC. We use the **Nostr strategy** (signaling via public Nostr relays) to
  avoid running any server and to dodge CORS/403 issues from torrent trackers.
- **localStorage** — host-side persistence of the whole session.

### How Trystero is loaded (important)

Trystero is imported via an **import map** in `<head>`:

```html
<script type="importmap">
  { "imports": { "trystero": "https://cdn.jsdelivr.net/npm/trystero@0.22.0/nostr/+esm" } }
</script>
```

Then `import { joinRoom } from 'trystero';` in the module script.

**Gotcha (already fixed):** the Nostr strategy is a *subpath export*
(`trystero/nostr`), not a root file. Paths like
`trystero@0.22.0/nostr.min.js` return **404** — that file does not exist in the
package. jsdelivr's `/+esm` endpoint bundles the ESM correctly. Do **not**
revert this to `nostr.min.js`.

---

## Running it

Must be served over `http(s)://` — **not** `file://` (import maps + WebRTC
require an origin). From the project folder:

```bash
npx serve .
```

- **Desktop:** `http://localhost:xxxx` is fine.
- **Mobile / other devices:** WebRTC generally needs **HTTPS** when the origin
  isn't `localhost`. Use a tunnel (`ngrok`, `cloudflared`) or an HTTPS static
  host to test on phones.

---

## Roles

Role is decided by the URL hash:

- **Host / Game Master (GM):** no `#room=...` in the URL, or `host=1` in the
  hash. A `roomId` is generated (`room-xxxxxxx`) and written to the hash as
  `#room=<roomId>&host=1`, so reloading the page keeps the host role. Sees the
  **Панель Ведущего** (host panel).
- **Player:** URL contains `#room=<roomId>&player=<name>`. Connects to the
  existing room as that named player. Sees a small status bar only.

The host generates per-player invite links and can copy them all at once to
paste into a group chat.

---

## Features

### Host (GM)
The panel is split into three `.section` cards: **1. Картинка карты**,
**2. Игроки и аватарки** (with "copy all links" at the top, above the player
list), **3. Комната** (recreate). The restore banner sits above them all.

- Set the map background two ways, both stored in `gameState.bgImage`:
  - **local file** (`<input type="file">` → `FileReader` → DataURL), or
  - **direct image URL** pasted into the URL field (Enter or "Загрузить по
    ссылке"). The URL is probed with an `Image()` first and rejected if it
    doesn't load; the URL itself is synced to peers, so it must be publicly
    reachable for every player.
  - `✕` clears the background.
  - A DataURL from a large file can exceed the `localStorage` quota — the save
    failure is caught and the host is told to use a URL instead. The map still
    works for the current session.
- Add players (name + optional avatar). Each player becomes a token. The avatar
  can be a pasted URL **or** a local file (`📁` → `FileReader` → DataURL); a
  typed URL wins over a previously picked file.
- Player rows are left-aligned: a fixed-width name column, then the avatar
  field ("Ссылка на аватарку"), `📁`, "Ссылка для входа" (copies the player's
  invite link) and delete. A file-based avatar shows the placeholder
  "Аватарка из файла" instead of the huge DataURL.
- Generate per-player links; "copy all links" button.
- Autosave the full session (map, tokens, avatars) to `localStorage` and a
  **restore banner** to resume a previous game on reload (rejoins the saved
  room, so old player links keep working).
- **🔄 Пересоздать комнату** — when the P2P connection goes stale and players
  can no longer join: leaves the current Trystero room, joins a brand-new
  `roomId`, and keeps the whole `gameState` (background image, avatars, token
  positions). New player links are regenerated and copied to the clipboard;
  the old links stop working, so they must be re-sent.
- Can drag **any** token.

- Panel can be hidden — see **Show/hide panel** below. It is force-opened on
  load when a saved game exists, so the restore banner isn't missed.

### Player
- Sees the current map + all tokens in real time.
- Can drag **only their own** token (the one whose name matches `player` in the
  URL). Other tokens are shown `.disabled`.
- Compact one-line header (`Комната: … | Вы: …`) instead of a full panel, and
  it can be hidden too.
- `body.role-player` makes the page a flex column of `100vh`, so `#board-wrap`
  fills everything left over after the body padding and the header — the map is
  effectively full-screen at all times.

### Show/hide panel (both roles)
- One button, two placements: while the panel is open it sits **in the flow**,
  in the panel's own top-left corner (32px, next to "Панель Ведущего" / the
  player status line); once the panel is hidden it becomes a 42px
  `position: fixed` button in the page's top-left corner over the map. The
  button is moved in the DOM (`btnSlot` ↔ `document.body`) and restyled via
  `body.panel-hidden`.
- Label: `⚙` when hidden, `✕` when shown. It toggles the panel belonging to the
  current role (host panel / player header).
- `body.panel-hidden` drops the body padding to 8px; for the host it also
  stretches `#board-wrap` to the full viewport height (the player layout
  already fills it via flex).
- The choice is remembered in `localStorage` (`p2p_tabletop_panel_hidden`).

### Token bounds
- Tokens are clamped to the board while dragging: `x` in `[0, boardWidth - 60]`,
  `y` in `[0, boardHeight - 60]` (board size = the map image's natural size, or
  the 500×500 CSS minimum when there's no map). Clamping happens on the
  dragger's own client, before the position is broadcast.
- New tokens are placed inside the bounds, and when the map image loads at a
  different size the host pulls stray tokens back in and re-broadcasts.

### Map view — zoom & pan (works on desktop + mobile)
- On-screen buttons (bottom-right): `−` / `+` / `⟲` reset + live % readout.
- Mouse wheel zoom (anchored to cursor).
- **Pinch-to-zoom** (two fingers), anchored to the pinch midpoint.
- **Pan** by dragging the map background (one finger / mouse).
- Zoom/pan is **view-only and local** — NOT part of `gameState`, not synced,
  not saved. Token coordinates live in the map's own coordinate space so every
  peer sees tokens in the same place regardless of personal zoom.

---

## Architecture / Data flow

### `gameState` (the only synced state)
```js
gameState = {
  bgImage: null,          // map image as a DataURL string
  tokens: {               // keyed by player name
    "Alice": { x, y, avatar }   // x/y in board (unscaled) pixels; avatar = URL|null
  }
}
```

### P2P channels (Trystero actions)
- `state` — full `gameState` snapshot. Host → players.
- `move` — a single token move `{ name, x, y }`. Any mover → everyone.

### Sync rules
- On `onPeerJoin`, the **host** pushes the full `gameState` to the new peer.
- `saveAndBroadcast()` (host only): writes `localStorage` **then**
  `sendState(gameState)` to everyone. Called after any host-side change.
- On drag end (`pointerup`), the mover sends a `move`; the host also
  `saveAndBroadcast()`s.

### Rendering & input
- `renderBoard()` rebuilds all tokens and sets the board background. Board size
  is set to the natural image size on `img.onload`.
- Dragging uses **Pointer Events** (`pointerdown/move/up/cancel`) so mouse +
  touch share one code path; `setPointerCapture` keeps tracking off-element.
- **Scale correction:** while dragging, screen delta is divided by `boardScale`
  so the token tracks the finger/cursor at any zoom.
- `#board` is transformed with `translate(panX,panY) scale(boardScale)`;
  `#board-wrap` is the fixed viewport (`overflow:hidden`, `touch-action:none`).
- Tokens have `touch-action:none`; a `pointerdown` on a token is ignored by the
  pan/pinch handler (`closest('.token')`) so dragging and panning don't fight.

---

## File map

- `index.html` — the entire app (markup, styles, and the ES module script).
  The `<!--version-->` comment at the top tracks iterations.
- `.idea/` — JetBrains project files (ignored for app purposes).
