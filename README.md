# NEXUS GRID — Mobile Browser Game Shell

A single-file HTML boilerplate for building mobile browser games. Drop in your game logic, wire it to the event bus, and ship.

---

## What it is

NEXUS GRID is a complete UI shell for portrait-mode mobile browser games. It provides all the structural chrome you need — system console, score bar, tab navigation, modals, settings, save slots — so you can focus entirely on your game logic.

Everything is a single `.html` file with no dependencies, no build step, and no framework.

---

## Structure

```
┌─────────────────────────────────┐
│  Console (system log)    20vh   │  ← nexus:log events appear here
├─────────────────────────────────┤
│  Score Bar               30px.  │  ← nexus:game:score events
├─────────────────────────────────┤
│  [TAB][TAB][TAB][TAB][TAB][TAB] │  ← tab bar, 58px
│                                 │
│  Game Panel (active tab)        │  ← your game UI lives here
│                                 │
└─────────────────────────────────┘
```

The toolbar in the console header contains: **⚙ Settings · 💾 Save · 📂 Load · ℹ About · ⛶ Fullscreen**

---

## Event Bus

All communication between your game and the shell goes through a tiny publish/subscribe bus exposed as the global `nexus` object.

```js
nexus.on('event:name', handler)   // subscribe
nexus.emit('event:name', data)    // publish
nexus.off('event:name', handler)  // unsubscribe
```

Your game should never touch shell DOM directly. Use events instead.

The bus is implemented as a factory function `createBus()`. The shell instantiates it once as `const nexus = createBus()`, but you can create additional isolated instances in tests without any cleanup overhead:

```js
// in your test suite
const bus = createBus(); // fresh instance, no shared state
bus.on('nexus:log', e => received.push(e));
bus.emit('nexus:log', { tag: 'ok', msg: 'test' });
```

Errors thrown inside a handler are caught and logged to the console (`[nexus] handler error on "event:name": ...`) without interrupting the other listeners registered on the same event.

### Full Event Reference

#### System
| Event | Payload | Description |
|---|---|---|
| `nexus:log` | `{ tag, msg }` | Write a line to the console. `tag`: `ok` · `inf` · `wrn` · `err` — note: use `wrn` (not `warn`) |
| `nexus:toast` | `{ msg, type, duration? }` | Show a temporary overlay notification. `type`: `ok` · `inf` · `warn` · `err` · `save` |
| `nexus:alert` | `{ msg, type }` | Show a blocking modal that requires user dismissal |
| `nexus:badge` | `{ tab, count }` | Set notification badge on a tab. `count: 0` removes it |
| `nexus:tab` | `{ id }` | Switch the active tab programmatically |

#### Game
| Event | Payload | Description |
|---|---|---|
| `nexus:game:start` | — | Signal game started (auto-logged) |
| `nexus:game:pause` | — | Signal game paused (auto-logged) |
| `nexus:game:over` | — | Signal game over (auto-logged) |
| `nexus:game:score` | `{ score?, level?, xp? }` | Update score bar values |

#### Audio
| Event | Payload | Description |
|---|---|---|
| `nexus:audio:play` | `{ src }` | Play a sound file. Respects audio enabled/volume settings |
| `nexus:audio:stop` | — | Stop current sound |
| `nexus:audio:volume` | `{ value }` | Set volume, range `0–1` |

#### Device
| Event | Payload | Description |
|---|---|---|
| `nexus:haptic` | `{ pattern }` | Vibrate. Accepts a duration in ms or an array of alternating vibrate/pause durations e.g. `[50, 30, 50]` |
| `nexus:net:online` | — | Emitted automatically when connection is restored |
| `nexus:net:offline` | — | Emitted automatically when connection is lost |
| `nexus:app:visible` | — | Emitted automatically when app returns to foreground |
| `nexus:app:hidden` | — | Emitted automatically when app is backgrounded |

#### Persistence
| Event | Payload | Description |
|---|---|---|
| `nexus:save` | `{ slot }` | Save state to slot 1–3 |
| `nexus:load` | `{ slot }` | Load state from slot 1–3 |

#### Settings
| Event | Payload | Description |
|---|---|---|
| `nexus:settings:changed` | `{ key, value }` | Emitted whenever a setting changes |

---

## How to Adapt

### 1. Replace the panels

Each tab panel is a `<div class="tab-panel" id="panel-{name}">` inside `#panels-container`. Replace the demo content with your own game UI. The panel id must match the `data-tab` attribute of the corresponding tab button.

```html
<button class="tab-button" data-tab="world">
  <span class="tab-icon">🌍</span>WORLD
</button>
...
<div class="tab-panel" id="panel-world">
  <!-- your game UI here -->
</div>
```

### 2. Wire your game loop to the bus

```js
// Log events
nexus.emit('nexus:log', { tag: 'ok', msg: 'Map loaded' });

// Update score
nexus.emit('nexus:game:score', { score: 1200, level: 3, xp: 450 });

// Show a toast
nexus.emit('nexus:toast', { msg: 'Enemy approaching!', type: 'warn' });

// Play a sound (respects user audio settings)
nexus.emit('nexus:audio:play', { src: './sounds/alert.mp3' });

// Vibrate on hit
nexus.emit('nexus:haptic', { pattern: [50, 30, 50] });
```

### 3. Add your game state to save slots

Find the `nexus:save` handler and add your serializable state:

```js
nexus.on('nexus:save', ({ slot }) => {
  const state = {
    // shell fields (keep these)
    activeTab: ...,
    highContrast: ...,
    savedAt: Date.now(),
    // your game state
    resources: myGame.getResources(),
    units: myGame.getUnits().map(u => u.serialize()),
  };
  localStorage.setItem(getSaveKey(slot), JSON.stringify(state));
  ...
});
```

Then restore it in the `nexus:load` handler:

```js
nexus.on('nexus:load', ({ slot }) => {
  const data = readSlot(slot);
  ...
  if (data.resources) myGame.setResources(data.resources);
  if (data.units)     myGame.loadUnits(data.units);
});
```

### 4. Remove demo content

Delete everything below the `NEXUS SHELL — DEMO CONTENT` comment block. The checklist in that section tells you exactly what to remove.

---

## Theming

### CSS Variables

All visual properties are driven by CSS variables defined in `:root`. Change them to reskin the entire shell instantly.

```css
:root {
  --color-accent:   #00ffe7;  /* primary accent (tabs, borders, highlights) */
  --color-danger:   #ff3e6c;  /* destructive actions, errors */
  --color-warning:  #ffe600;  /* warnings, resources, scores */
  --color-bg:       #0a0c10;  /* page background */
  --color-panel:    #0f1218;  /* card / panel background */
  --color-border:   #1e2530;  /* borders and dividers */
  --color-text:     #c8d6e5;  /* body text */
  --color-dim:      #4a5568;  /* muted / secondary text */
}
```

### Runtime Color Picker

The Settings panel exposes three color pickers (accent, danger, warning) that update the CSS variables live and persist to `localStorage` under `nexus-cfg`.

### High Contrast Mode

A High Contrast toggle in Settings switches the shell to maximum-readability mode (yellow on black). User preference is persisted. When HC is off, the custom colors from the color pickers are automatically restored.

---

## Settings & Persistence

User preferences are saved to `localStorage` under the key `nexus-cfg`:

| Key | Type | Default | Description |
|---|---|---|---|
| `audioEnabled` | boolean | `true` | Master audio switch |
| `volume` | number 0–100 | `70` | Master volume |
| `vibroEnabled` | boolean | `true` | Haptic feedback |
| `highContrast` | boolean | `false` | High contrast mode |
| `accentColor` | hex string | `#00ffe7` | Primary accent color |
| `dangerColor` | hex string | `#ff3e6c` | Danger color |
| `warningColor` | hex string | `#ffe600` | Warning color |

Game save slots are stored under `nexus-save-1`, `nexus-save-2`, `nexus-save-3`.

---

## Browser Support

Requires a modern mobile browser (Chrome 105+, Safari 15.4+, Firefox 110+) for full support of:

- `100dvh` dynamic viewport units
- `color-mix()` CSS function
- `navigator.vibrate()` (Chrome/Android only; gracefully ignored on iOS)
- `requestFullscreen()` (behavior varies by browser/OS)

---

## File Layout

```
mobile-tabs.html   (1887 lines)
│
├── <style>
│   ├── CSS Variables (:root)
│   ├── High Contrast theme (:root.hc)
│   ├── Core layout (body, console, score bar, game area, tab bar)
│   ├── UI Components (card, stat-grid, progress, map, chat, leaderboard, shop)
│   └── Overlays (toast, bottom sheet, settings, slot picker, about, landscape warning)
│
├── <body>
│   ├── #landscape-warning
│   ├── #shell-console (toolbar + log lines)
│   ├── #score-bar
│   └── #game-area
│       ├── #toast-container
│       ├── #modal-settings
│       ├── #modal-slots
│       ├── #modal-about
│       ├── #tab-bar
│       └── #panels-container
│           └── .tab-panel × 6 (demo content)
│
└── <script>
    ├── Event Bus (nexus object)
    ├── Settings & Persistence
    ├── Haptic handler
    ├── Audio handler
    ├── Console log handler
    ├── Toast handler
    ├── Tab switching handler
    ├── Badge handler
    ├── Score bar handler
    ├── Game lifecycle handlers
    ├── Alert handler
    ├── Save/Load slot system
    ├── Modal management
    ├── Toolbar button wiring
    ├── Settings panel controls
    ├── Device & app lifecycle (online/offline, visibilitychange)
    └── DEMO CONTENT (delete this section)
```

---

## License

MIT — do whatever you want with it.
