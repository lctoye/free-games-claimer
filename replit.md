# free-games-claimer

## Overview
A Node.js CLI automation tool that uses Playwright (headless Firefox) to automatically claim free games on:
- Epic Games Store
- Amazon Prime Gaming
- GOG
- Unreal Engine (Assets)
- AliExpress (experimental)
- Steam (experimental)

## Project Structure
- `epic-games.js` - Epic Games Store claimer
- `prime-gaming.js` - Amazon Prime Gaming claimer
- `gog.js` - GOG claimer
- `unrealengine.js` - Unreal Engine Assets claimer
- `aliexpress.js` - AliExpress claimer (experimental)
- `steam-games.js` - Steam claimer (experimental)
- `src/config.js` - Configuration via environment variables
- `src/util.js` - Shared utilities (DB, browser helpers, etc.)
- `src/migrate.js` - Data migration utilities
- `data/` - Runtime data (browser profiles, JSON databases, screenshots)

## Configuration
All configuration is done via environment variables, loaded from `data/config.env`. Key variables:
- `EG_EMAIL`, `EG_PASSWORD`, `EG_OTPKEY` - Epic Games credentials
- `PG_EMAIL`, `PG_PASSWORD`, `PG_OTPKEY` - Prime Gaming credentials
- `GOG_EMAIL`, `GOG_PASSWORD` - GOG credentials
- `SHOW=1` - Show browser window
- `DEBUG=1` - Enable debug mode
- `DRYRUN=1` - Dry run (don't actually claim)

## Runtime
- **Language**: Node.js (ESM modules, requires Node >= 17)
- **Package manager**: npm
- **Browser**: Playwright Firefox (headless by default)
- **Database**: lowdb (JSON files in `data/`)

## Workflow
- **Start application**: `bash -c "node prime-gaming.js; node gog.js; echo sleeping; sleep 1d"` (console output, checks Prime Gaming and GOG)

## Environment Variables (configured)
- `EMAIL`, `EG_EMAIL`, `GOG_EMAIL` = 2ChrisOrr@gmail.com
- `PG_EMAIL` = chrisorr@email.com
- `NOTIFY_TITLE` = Free Games Claimer
- Secrets: `EG_PASSWORD`, `PG_PASSWORD`, `GOG_PASSWORD`, `NOTIFY` (Pushover)

## System Dependencies
- firefox, xvfb-run, dbus, gtk3, glib, nss, alsa-lib, libdrm, mesa
- xorg.libxcb, xorg.libX11, xorg.libXext, xorg.libXrandr, xorg.libXcomposite
- xorg.libXcursor, xorg.libXdamage, xorg.libXfixes, xorg.libXi
- pango, atk, cairo, gdk-pixbuf, freetype, fontconfig, xorg.libXrender
- gcc-unwrapped

## Notes
- This is a pure CLI tool with no web frontend
- Login credentials need to be set as environment variables or in `data/config.env`
- On first run, the script waits for you to log in via the browser
- Browser profiles are saved to `data/browser/` to avoid re-logging in
