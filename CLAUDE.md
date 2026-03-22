# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project overview
- `MouseInc.Settings` is the Vue 3 + TypeScript settings UI for the MouseInc desktop app.
- The UI is not standalone in normal use: it talks to a local MouseInc process over WebSocket and edits MouseInc JSON config data.

## Common commands
```bash
# install deps
npm install

# start dev server (opens browser)
npm run dev

# production build
npm run build

# preview built output
npm run preview

# lint all source files
npm run lint

# lint with autofix
npm run lint:fix

# lint a single file (closest equivalent to a single-test workflow)
npx eslint src/view/switch.vue
```

### Tests
- There is currently no automated test runner configured in this repository (`package.json` has no `test` script, and there are no project test files under `src/`, `test/`, or `tests/`).
- Validation is currently lint + manual UI verification.

## Runtime/development integration details
- MouseInc can open the localhost dev settings page when launching settings while holding `SHIFT`.
- WebSocket endpoint is built in `src/components/main/main.vue` as `ws://127.0.0.1:{port}/ws`.
  - `port` comes from URL query param `port`; falls back to `80`.
- Main message flow:
  - on connect: send `{"type":"load_settings"}`
  - on save: send `{"type":"save_settings","config":"<formatted JSON string>"}`
  - on reset: send `{"type":"reset_settings"}`

## High-level architecture

### 1) App bootstrap
- `src/main.ts` creates app, router, Vuex store, and i18n.
- Language is normalized from localStorage key `local` (`zh-CN` / `zh-TW` / `en-US`) with fallback to `zh-CN`.

### 2) Shell/layout and app lifecycle
- `src/components/main/main.vue` is the primary shell:
  - sidebar + breadcrumb + header actions + page content
  - WebSocket connection lifecycle
  - save/reset/refresh actions
  - `modified` state detection and save button enablement

### 3) Router-driven settings sections
- Route definitions live in `src/router/routers.ts`.
- Main groups:
  - `/` (switches/exclusions)
  - `/gesture` (gesture behavior, gesture list, global/custom matches, demo)
  - `/other` (edge/corner/copy/hotkey/keycast/i18n)
- `src/router/index.ts` adds loading overlay and updates page title after navigation.

### 4) Central state model (Vuex)
- `src/store/index.ts` holds:
  - `cfg`: current MouseInc config object
  - `gestures`: gesture preview map (sign -> image)
  - `local`: current UI locale
  - `breadCrumbList`
- `setSettings` performs compatibility normalization (notably defaulting `OcrService` and `HotCorner`) and injects `WheelSwitchUp/Down` from `placeholder` gesture assets.
- Many pages edit nested `cfg` directly; this is an existing pattern in this codebase.

### 5) Settings pages and shared editors
- Pages under `src/view/*.vue` each edit different config domains (`switch`, `settings`, `list`, `match`, `custom_match`, etc.).
- `src/view/components/json.vue` is a key shared action editor:
  - dual UI/raw modes
  - JSON repair via `jsonrepair`
  - formatting via `js-beautify`

### 6) Menu/title/i18n helpers
- `src/libs/util.ts` builds menu/breadcrumb from routes and resolves translated titles.
- Locale files are in `src/locale/lang/*.js`.
- `src/config/index.ts` enables i18n (`useI18n: true`) and defines default home route name (`switch`).

### 7) Styling system
- `src/index.less` defines global design tokens and system dark/light theme via `prefers-color-scheme`.
- `src/styles/common.less` provides shared page/card/table/dialog utility styles used across view pages.

### 8) Build/deploy characteristics
- `vite.config.js` key behavior:
  - base path: `/mouseinc/`
  - aliases: `@ -> src`, `_c -> src/components`
  - auto-import/plugins for Vue, Vue Router, Vuex, Vue I18n, and Element Plus
  - CSS injected by JS (`vite-plugin-css-injected-by-js`) and `cssCodeSplit: false`
  - hashed output filenames with manual `vendor` chunk
- CI workflow: `.github/workflows/build-and-package.yml` runs on `main` push/PR, uses Node 22, and executes `npm install` + `npm run build`.

## Source-of-truth references for config semantics
- `docs/docs/config.md` documents MouseInc config keys and action structure (`MouseGesture`, `MatchGlobal`, `MatchCustom`, `Hotkeys`, `WheelEdge`, `HotCorner`, etc.).
- When uncertain about a field shape, align with `docs/docs/config.md` and current runtime usage in `src/view/*` + `src/store/index.ts`.