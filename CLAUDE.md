# NN-WHNVR — Claude Code Context

Interactive installation with audience-submitted messages displayed as live visuals, controlled remotely via Firebase Realtime Database.

## Project Structure

```
nn-whnvr/
├── submit.html      # Audience-facing input page (phones/tablets)
├── curtain.html     # Visual 1: physics-based beaded curtain
├── panel.html       # Remote control panel for Winnie
├── tank.html        # Visual 2: fish tank (Winnie's, in progress)
└── display.html     # Switcher page shown on projector (TODO)
```

## Architecture

All pages communicate exclusively through **Firebase Realtime Database** — no direct page-to-page communication.

```
Audience (submit.html)
    └── writes → Firebase db/notes/{id}: { text, createdAt }

Winnie (panel.html)
    └── writes → Firebase db/settings/{ beadRadius, driftSpeed, strandSpacing, palette, activeVisual }

Projected screen (curtain.html / tank.html)
    └── reads ← Firebase db/notes       (renders messages as visuals)
    └── reads ← Firebase db/settings    (applies Winnie's live adjustments)
```

## Firebase Config

```js
const firebaseConfig = {
  apiKey: "AIzaSyAPA5EGbTXMnRJ5uX4O3JvJdCrtSHr5LHQ",
  authDomain: "nn-whnvr.firebaseapp.com",
  databaseURL: "https://nn-whnvr-default-rtdb.asia-southeast1.firebasedatabase.app",
  projectId: "nn-whnvr",
  storageBucket: "nn-whnvr.firebasestorage.app",
  messagingSenderId: "82298055324",
  appId: "1:82298055324:web:2380ad13ff2bd3e7591b59"
};
```

Firebase SDK version: **12.2.1** (loaded via CDN, no npm build step).

## Firebase Data Schema

```
/notes
  /{pushId}
    text: string
    createdAt: serverTimestamp

/settings
  activeVisual: "curtain" | "tank"
  beadRadius: number          (default 14)
  driftSpeed: number          (default 0.3)
  strandSpacing: number       (default 60)
  palette: "candy" | "neon" | "pastel" | "fire" | "ocean" | "mono"
```

## curtain.html — Key Details

Physics engine: custom Verlet integration (no external library).  
No build step — pure ES module HTML file, works with Live Server.

**CONFIG object** (live-updatable from Firebase settings):
```js
const CONFIG = {
  BEAD_RADIUS:    14,   // circle size in px
  BEAD_SPACING:   36,   // vertical distance between beads
  STRAND_SPACING: 60,   // horizontal distance between strands
  DRIFT_SPEED:    0.3,  // px/frame conveyor speed
};
```

**Key behaviours:**
- On load: fetches all existing `/notes` via `get()`, spreads them evenly across screen width
- New messages: `onChildAdded` fires → new strand enters from right edge
- Conveyor loop: strands drift left continuously; when a pin goes past `-STRAND_SPACING * 2`, it teleports to just right of the rightmost strand
- Mouse repulsion: radius 90px, strength 6 — cursor hidden via `cursor: none`
- Palette changes from panel update existing strand colors on next render (beads are re-rendered from cache keyed by `char + color + beadRadius`)

**Classes:** `Vec2`, `Particle`, `Constraint`  
**Draw:** offscreen canvas per character (pre-rendered circles with char inside), rotated to follow strand angle via `ctx.setTransform`

## panel.html — Key Details

Winnie opens this remotely. All controls write to `db/settings/` via `set()`.  
Also reads `db/notes` via `onChildAdded` to show a live message feed.  
On load, syncs UI state from Firebase via `onValue(ref(db, 'settings'))` so refreshing preserves her last settings.

**Controls:**
- Active visual switch (curtain / tank)
- Bead size slider (6–28px)
- Drift speed slider (0–2.0 px/frame)
- Strand spacing slider (30–120px)
- Palette picker (6 presets)
- Live incoming message feed with count

## Palette Presets

```js
candy:  ['#FF6B9D','#FF9F4A','#FFE14A','#6BFF9D','#4AF0FF','#9D6BFF', ...]
neon:   ['#FF0080','#00FFCC','#FFFF00','#FF6600','#00CCFF','#CC00FF', ...]
pastel: ['#FFB3C6','#FFD9A0','#FFFACD','#B3F0D4','#B3E5FF','#D4B3FF', ...]
fire:   ['#FF2200','#FF6600','#FF9900','#FFCC00','#FF4400','#FF8800', ...]
ocean:  ['#00CFFF','#0099CC','#006699','#00FFCC','#33BBFF','#0044AA', ...]
mono:   ['#FFFFFF','#DDDDDD','#BBBBBB','#999999','#EEEEEE','#CCCCCC', ...]
```

## TODO

- [ ] `tank.html` — Winnie's fish tank visual (reads same `/notes` and `/settings`)
- [ ] `display.html` — switcher page for projector; shows curtain or tank based on `settings/activeVisual`, crossfades between them
- [ ] Host on GitHub Pages so Winnie can access `panel.html` remotely
- [ ] Tighten Firebase security rules before event (currently open read/write for dev)
- [ ] Add memory cleanup for strands that have looped many times (performance)

## Firebase Security Rules (current — dev only)

```json
{
  "rules": {
    ".read": true,
    ".write": true
  }
}
```

Tighten before event: allow public write to `/notes`, read-only on `/settings` for visuals, write on `/settings` only for panel (consider auth or obscurity).

## Development

No build step. Open files directly with VS Code Live Server.  
All files are standalone HTML — CSS and JS inline, Firebase loaded via CDN.

To test: open `submit.html` in one tab, `curtain.html` in another.  
Submit a message → should appear as a new strand within ~1 second.
