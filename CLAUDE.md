# Hydro – Wasser-Erinnerung

## Project Overview

**Hydro** (v2.8) is a German-language water intake reminder PWA. The entire application lives in a single HTML file — no build step, no dependencies, no package manager. It runs directly in any modern browser and can be installed as a Progressive Web App on iOS and Android.

## Repository Structure

```
Wasserreminder/
├── wasser-reminder.html   # Entire application (HTML + CSS + JS, ~2,230 lines)
├── icon.png               # 180×180 Apple touch icon
├── icon-192.png           # 192×192 PWA icon
├── icon-512.png           # 512×512 PWA icon (maskable)
└── CLAUDE.md              # This file
```

There is no `package.json`, Makefile, or CI configuration. All code is inline.

## Tech Stack

| Layer | Technology |
|-------|-----------|
| Language | Vanilla JavaScript (ES6+), CSS3, HTML5 |
| Storage | `localStorage` (key: `hydro_root`) |
| Offline | Inline Service Worker |
| Weather | Open-Meteo API (no auth required) |
| Audio | HTML5 `<audio>` + Web Audio API fallback |
| Fonts | Google Fonts — `Fraunces` (display) + `Inter` (body) |
| Build | None — open the file directly |

## Running the App

```bash
# Option 1: open directly
open wasser-reminder.html

# Option 2: serve for full PWA features (notifications, service worker)
npx serve .
# or
python3 -m http.server 8080
```

Full PWA capabilities (notifications, service worker caching, install prompt) require HTTPS or `localhost`.

## Architecture

The file is organized in three top-level sections:

1. **`<style>`** (lines ~23–780) — CSS custom properties for theming, all component styles, animations.
2. **`<body>`** (lines ~780–1,100) — Static HTML scaffolding; the JS re-renders inner content dynamically.
3. **`<script>`** (lines ~1,100–2,230) — All application logic.

### State Model

All runtime state is derived from one root object persisted under `localStorage.hydro_root`:

```js
root = {
  activeProfile: "default",       // key into root.profiles
  theme: "ocean",                  // ocean | sunset | forest | berry | custom
  mode: "auto",                    // auto | dark | light
  customAccent: "#9c27b0",
  profiles: {
    default: {
      name: "...", avatar: "💧",
      baseGoal: 2000,              // ml
      dailyLog: { "YYYY-MM-DD": [{ ml, type, hydration, time }] },
      lifetimeTotal: 0,            // ml
      lifetimeDays: 0,
      streak: 0, bestStreak: 0,
      achievements: {},
      // settings: soundOn, hapticOn, reminderOn, weatherAdjust, ...
    }
  }
}
```

`root.profiles[root.activeProfile]` is the **active profile** — aliased as `state` throughout the JS.

### Key Functions

| Function | Purpose |
|----------|---------|
| `loadRoot()` | Deserialise + migrate from `localStorage` |
| `save()` | Persist `root` back to `localStorage` |
| `render()` | Re-render the entire main UI |
| `applyTheme()` | Sync `data-theme` / `data-mode` on `<html>` |
| `add(ml, type)` | Log a drink entry and trigger all side-effects |
| `addCustom()` | Log from the custom-amount input |
| `rolloverDay()` | Reset daily stats when the date changes |
| `refreshWeather()` | Fetch weather and update goal boost |
| `checkAchievements()` | Evaluate & unlock achievement badges |
| `openSettings()` / `openStats()` / `openLog()` | Show modal panels |
| `exportBackup()` / `importBackup()` | JSON data backup/restore |
| `exportICS()` | Generate `.ics` calendar reminder file |
| `startReminder()` | Set up in-app notification interval |
| `showToast(msg)` | Display transient notification strip |

### Drink Types & Hydration Factors

Defined in the `DRINK_TYPES` constant. Each entry maps a drink key to `{ label, icon, factor }`. `factor` is multiplied against logged `ml` to calculate net hydration (e.g., coffee = 0.80, beer = 0.60, water = 1.00).

### Themes

Five built-in themes (`ocean`, `sunset`, `forest`, `berry`, `custom`) plus dark/light/auto mode. Themes are applied via `data-theme` and `data-mode` attributes on `<html>`. All colors are CSS custom properties (`--accent`, `--bg`, `--surface`, `--text`, etc.) — never hard-coded hex values in component styles.

### Achievements

20 achievements defined in the `ACHIEVEMENTS` array. Each has `{ id, name, icon, desc, check(state) }`. `check` is called inside `checkAchievements()` after every `add()`.

## Development Conventions

### Editing the single file

- **CSS first**: all visual changes belong in the `<style>` block.
- **JS constants at the top of `<script>`**: `DRINK_TYPES`, `THEMES`, `AVATARS`, `ACHIEVEMENTS`, `TIPS` are declared before any functions.
- **No modules, no bundler** — everything is in global scope. Keep new helpers near related functions.
- **`render()` is idempotent** — it replaces `.innerHTML` of key containers. Never store references to rendered DOM nodes across renders.
- **Always call `save()` after mutating `root` or `state`.**
- **Date key format**: `YYYY-MM-DD` via `getToday()`. Never use `new Date().toLocaleDateString()` for keys.

### Localization

The UI is German only (`lang="de"`). All user-visible strings are German. Keep new strings in German.

### CSS Custom Properties

Use existing variables — never introduce new hex colors directly in component rules. Available accent variables: `--accent`, `--accent2`, `--accent3`, `--accent-rgb`. Surface variables: `--bg`, `--bg2`, `--surface`, `--surface2`, `--glass`, `--glass-strong`, `--glass-border`. Text: `--text`, `--muted`, `--strong`.

### No External Dependencies

Do not add `npm install`, CDN `<script>` tags, or any external JS library. The zero-dependency constraint is intentional.

### iOS PWA Compatibility

Several workarounds exist specifically for iOS Safari in PWA mode:
- Font size forced to `16px` on inputs to prevent auto-zoom.
- Audio playback uses `<audio id="plopAudio">` with a Web Audio API fallback.
- `maximum-scale=1.0, user-scalable=no` on the viewport meta.
- `env(safe-area-inset-*)` for notch/home-bar padding.

Do not break these when editing layout or audio code.

### LocalStorage Keys

| Key | Contents |
|-----|----------|
| `hydro_root` | Full application state (profiles, settings, log) |
| `hydro_hint_dismissed` | `"1"` when the PWA install hint has been closed |
| `hydro_state` | Legacy single-profile format — only read for migration |

## Git Workflow

- Default development branch: `main`
- All code lives in `wasser-reminder.html` — changes are a single-file diff.
- Commit messages should be descriptive (the upload history of "Add files via upload" is not a model to follow).
- There is no CI pipeline; test manually in a browser before committing.
