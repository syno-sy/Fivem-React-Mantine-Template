# FiveM React + Mantine Template

A FiveM resource boilerplate that pairs a **React 19** + **Mantine 9** NUI with **Lua** client/server scripts. It includes visibility handling, locale loading, NUI callbacks, and browser-based development with mocked events.

Based on [fivem-react-boilerplate-lua](https://github.com/project-error/fivem-react-boilerplate-lua).

## Features

- **React 19** + **TypeScript** + **Vite** for fast dev and optimized production builds
- **Mantine 9** UI components and theming
- **ox_lib** integration for locales and notifications
- **ESX** example: fetch player money from the client
- **VisibilityProvider** — show/hide NUI with focus and Escape/Backspace to close
- **LocaleProvider** — JSON locale files loaded from Lua
- **useNuiEvent** — listen for `SendNUIMessage` actions in React
- **fetchNui** — call `RegisterNUICallback` from React (with browser mocks)
- **debugData** — emulate NUI messages when developing in the browser
- Hot in-game builds (`start:game`) — restart the resource to see changes

## Requirements

- [Node.js](https://nodejs.org/) **v20.11+** (CI uses v24)
- [pnpm](https://pnpm.io/) (recommended; used by CI)
- [ox_lib](https://github.com/overextended/ox_lib) — must start **before** this resource
- [es_extended](https://github.com/esx-framework/esx_core) — used in the example callbacks (adapt or remove if you use another framework)

A basic understanding of modern web development (React, npm/pnpm scripts) is helpful.

## Project structure

```
├── client/
│   ├── init.lua          # Locale loading NUI callback
│   └── client.lua        # UI open/close, ESX money, config callback
├── server/
│   └── server.lua        # ESX shared object (extend as needed)
├── shared/
│   └── config.lua        # Theme colors, default language
├── locales/
│   └── en.json           # UI strings (add more languages as needed)
├── fxmanifest.lua
└── web/                  # React app (build output → web/build/)
    ├── src/
    │   ├── components/   # UI components
    │   ├── hooks/        # useNuiEvent
    │   ├── providers/    # Visibility, Locale
    │   ├── theme/        # Mantine theme
    │   └── utils/        # fetchNui, debugData, misc
    └── package.json
```

## Getting started

1. Clone this repo or use it as a template.
2. Place the folder inside your server's `resources` directory.
3. Ensure **ox_lib** and **es_extended** (or your framework) are installed and started before this resource.
4. Build the UI (see [Installation](#installation)).
5. Add `ensure your-resource-name` to `server.cfg`.
6. In-game, run `/open-ui` to open the example UI.

## Installation

From the `web` folder:

```sh
pnpm install
```

### Browser development

Runs Vite dev server with hot reload. The UI is visible in the browser with a background image; `debugData` simulates NUI messages.

```sh
pnpm dev
```

### Hot builds in-game

Writes the build to disk on every change. Restart the resource in-game to pick up updates (no full rebuild step each time).

```sh
pnpm run start:game
```

### Production build

Required before shipping. Output goes to `web/build/` (referenced by `fxmanifest.lua`).

```sh
pnpm run build
```

### Other scripts

```sh
pnpm run typecheck   # TypeScript check without emit
pnpm run preview     # Preview production build locally
pnpm run prettier    # Check formatting
```

## Configuration

Edit `shared/config.lua`:

```lua
Config = {}

Config.PrimaryColor = 'blue'   -- Mantine color name (used by getConfig callback)
Config.PrimaryShade = 8        -- Mantine shade 0–9
Config.language = 'en'         -- Locale file: locales/<language>.json
```

The client exposes `getConfig` via NUI callback; wire it in React (e.g. in `theme.ts` or `main.tsx`) if you want dynamic Mantine colors from Lua.

## Localization

1. Add keys to `locales/en.json` (and other language files).
2. Match keys in `web/src/providers/LocaleProvider.tsx` (`Locale` interface).
3. Use `useLocales()` in components:

```tsx
const { locale } = useLocales();

<Button>{locale.ui_buttonText}</Button>
```

On mount, the UI calls `loadLocale`; Lua loads the JSON and sends it back with `setLocale`.

**Add a new language:** create `locales/de.json`, set `Config.language = 'de'`.

## In-game usage

| Action | How |
|--------|-----|
| Open UI | `/open-ui` or trigger `OpenNui(true)` from your Lua |
| Close UI | Close button, or **Escape** / **Backspace** (calls `hide-ui`) |

## NUI communication

### Lua → React (`SendNUIMessage`)

Client example (visibility):

```lua
SetNuiFocus(true, true)
SendNUIMessage({
  action = 'setVisible',
  data = true
})
```

Locale example (`client/init.lua`):

```lua
SendNUIMessage({
  action = 'setLocale',
  data = json.decode(JSON)  -- table from locales/en.json
})
```

React listens with **useNuiEvent**:

```tsx
import { useNuiEvent } from '../hooks/useNuiEvent';

useNuiEvent<boolean>('setVisible', (visible) => {
  console.log('UI visible:', visible);
});

useNuiEvent<Locale>('setLocale', (data) => {
  setLocale(data);
});
```

### React → Lua (`fetchNui` + `RegisterNUICallback`)

React:

```tsx
import { fetchNui } from '../utils/fetchNui';

// No body
await fetchNui('hide-ui');

// With data
const money = await fetchNui('getPlayerMoney');

// Browser dev: return mock when not in CEF
const config = await fetchNui('getConfig', undefined, {
  primaryColor: 'blue',
  primaryShade: 8,
});
```

Lua:

```lua
RegisterNUICallback('hide-ui', function(_, cb)
  SetNuiFocus(false, false)
  SendNUIMessage({ action = 'setVisible', data = false })
  cb({})
end)

RegisterNUICallback('getPlayerMoney', function(data, cb)
  local money = ESX.PlayerData.money
  cb(money)
end)

RegisterNUICallback('getConfig', function(_, cb)
  cb({
    primaryColor = Config.PrimaryColor,
    primaryShade = Config.PrimaryShade,
  })
end)
```

### Browser development (`debugData`)

Emulates `SendNUIMessage` in the browser (development mode only):

```tsx
import { debugData } from '../utils/debugData';

debugData([
  {
    action: 'setVisible',
    data: true,
  },
], 1000);  // delay in ms

debugData([
  {
    action: 'setLocale',
    data: {
      ui_playerMoney: 'Player Money',
      ui_buttonText: 'Click to Get Player Money',
      ui_reset: 'reset',
    },
  },
], 2000);
```

`main.tsx` already calls `debugData` for `setVisible` so the UI appears when running `pnpm dev`.

### Environment check

```tsx
import { isEnvBrowser } from '../utils/misc';

if (isEnvBrowser()) {
  // Running in normal browser, not FiveM CEF
}
```

## Providers

### VisibilityProvider

Mount at the app root (see `main.tsx`). Listens for `setVisible`, toggles CSS visibility, and closes on Escape/Backspace via `hide-ui`.

```tsx
import { VisibilityProvider, useVisibility } from './providers/VisibilityProvider';

// In root
<VisibilityProvider>
  <App />
</VisibilityProvider>

// In a child (optional)
const { visible, setVisible } = useVisibility();
```

### LocaleProvider

Loads locale on mount, listens for `setLocale`.

```tsx
import LocaleProvider, { useLocales } from './providers/LocaleProvider';

<LocaleProvider>
  <App />
</LocaleProvider>
```

## Example UI (`AppComp.tsx`)

The sample component demonstrates:

- **Close** — `fetchNui('hide-ui')`
- **Get money** — `fetchNui('getPlayerMoney')` with fallback mock on error in browser
- **Reset** — local state only
- **Locales** — labels from `useLocales()`
- **Mantine** — `Button`, `Paper`, `ActionIcon`, Iconify icons

## Mantine theming

Edit `web/src/theme/theme.ts`:

```ts
import { createTheme } from '@mantine/core';

export const theme = createTheme({
  primaryColor: 'blue',
  // ...other Mantine theme overrides
});
```

Global styles are imported in `main.tsx`: `@mantine/core/styles.css`.

## fxmanifest.lua

Key entries:

- `ui_page 'web/build/index.html'` — requires a production (or watch) build
- `shared_scripts` — `@ox_lib/init.lua`, `shared/**/*`
- `files` — `locales/*.json`, `web/build/**/*`

Supports **GTA5** and **RDR3** (`fx_version "cerulean"`).

## CI

GitHub Actions (`.github/workflows/ci.yml`) runs on push/PR:

- `pnpm install --frozen-lockfile` in `web/`
- `pnpm build`

## Troubleshooting

| Issue | Check |
|-------|--------|
| Blank UI in-game | Run `pnpm run build` or `pnpm run start:game`; ensure `web/build/` exists |
| `ox_lib should be started before this resource` | Start ox_lib before this resource in `server.cfg` |
| Locale not found | File exists at `locales/<Config.language>.json`; falls back to `en.json` |
| NUI callbacks fail in browser | Use `mockData` third argument on `fetchNui` or `debugData` for events |
| ESX errors | Resource expects `es_extended`; remove or replace ESX calls if not using ESX |

## Credits

- [project-error/fivem-react-boilerplate-lua](https://github.com/project-error/fivem-react-boilerplate-lua) — NUI patterns and utilities
- [Mantine](https://mantine.dev/) — React UI library
- [overextended/ox_lib](https://github.com/overextended/ox_lib) — Lua utilities and locale helpers

## License

See [LICENSE](LICENSE) in this repository.
