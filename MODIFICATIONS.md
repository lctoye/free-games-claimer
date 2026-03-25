# Modifications from Upstream Dev Branch

This document tracks all changes made to the `free-games-claimer` fork relative to the **dev branch** of [vogler/free-games-claimer](https://github.com/vogler/free-games-claimer), for potential contribution back upstream.

The upstream dev branch already included the switch from `playwright-firefox` to `playwright-chromium`, the `fingerprint-injector` integration, `src/epic-games-mobile.js`, and updated dependencies (`dotenv`, `lowdb`, `eslint`). Those are **not** listed here — only our additions and changes are documented.

---

## Dead Code Cleanup

### Scripts removed
- `aliexpress.js` — AliExpress coin collector (experimental, unused)
- `steam-games.js` — Steam games library scraper (not a game claimer)
- `unrealengine.js` — Unreal Engine asset claimer (duplicate of epic-games.js logic, WIP)
- `src/migrate.js` — One-time data migration script (no longer needed)
- `src/version.js` — Unused version utility

### Dead code removed from `src/util.js`
- `stealth()` function (~35 lines) — Puppeteer stealth evasions injected via `addInitScript`. No longer needed since patchright has built-in anti-detection.
- `launchChromium()` function (~20 lines) — Wrapper for launching Chromium persistent context. Defined but never called; each script launches its own context directly.

### Dead config removed from `src/config.js`
- `ae_email` and `ae_password` — AliExpress credentials for the deleted script.

---

## Browser Engine: playwright-chromium → patchright

The upstream dev branch uses `playwright-chromium`. We replaced it with [`patchright`](https://github.com/nicbarker/patchright), a Chromium fork with built-in stealth/anti-detection that eliminates the need for separate fingerprint injection.

### Changes
- `package.json`: Replaced `playwright-chromium` dependency with `patchright`
- All scripts: Changed `import { chromium } from 'playwright-chromium'` to `import { chromium } from 'patchright'`
- `src/util.js`: Removed `stealth()` function (patchright handles this natively)
- Removed stale commented-out `playwright-firefox` imports from all scripts

### Note
- Browser profiles are not compatible between `playwright-chromium` and `patchright` — sessions in `data/browser/` must be re-established

---

## Bug Fixes

### `prime-gaming.js` — Login error detection (upstream bug)
- Changed `error.trim.length` to `error.trim().length`
- `.trim` without `()` is a function reference (arity 0), so `.length` always returned 0
- The check `!error.trim.length` was therefore always truthy, meaning login errors (wrong password, account locked, etc.) were never detected or reported

### Notification reliability
- Added `await` to all `notify()` calls in `catch` and `finally` blocks across all 3 original claimer scripts
- Without `await`, the Node.js process could exit before the apprise notification was sent
- Affected files: `epic-games.js` (2 calls), `prime-gaming.js` (2 calls), `gog.js` (2 calls)

### `epic-games.js` — Removed noisy "already in library" notification
- Removed `notify()` call for games already in the user's Epic library
- The console log still shows "already in library" but no push notification is sent

### GOG selector fix
- GOG changed their page structure — `#menuUsername` is no longer reliably present
- Login detection loop changed from `while (signIn visible AND username not visible)` to `while (not logged in)` with multiple fallback selectors
- Username selector broadened: `#menuUsername, [hook-test="menuUsername"], .menu-username`
- Username reading wrapped in try/catch with 10s timeout — falls back to `'unknown'`
- Added 3s wait after page load to let GOG's Angular app hydrate before checking selectors

---

## Steam Free-to-Keep Game Claimer

### New file: `steam.js`
Automatically discovers and claims temporarily free games on Steam (100% off promotions for normally-paid games). Does NOT claim permanently free-to-play games or free weekend trials.

### Discovery
- Uses SteamDB's curated free promotions page (`steamdb.info/upcoming/free/`)
- SteamDB pre-separates "Free to Keep" promotions from "Free Weekend" / "Play for Free" events
- Extracts app ID, game name, and promotion end date from each entry
- Visits each game's Steam store page for rating, price, ownership check, and claiming

### Quality filtering
- `STEAM_MIN_RATING` (default: 6 = Mostly Positive) — Rating scale 1-9:
  - 9: Overwhelmingly Positive, 8: Very Positive, 7: Positive, 6: Mostly Positive
  - 5: Mixed, 4: Mostly Negative, 3: Negative, 2: Very Negative, 1: Overwhelmingly Negative
- `STEAM_MIN_PRICE` (default: 10 = $10 USD) — Minimum original price to filter out cheap/shovelware titles
- Games with no reviews are skipped (cannot verify quality)

### Claiming flow
- Handles Steam age verification gates (date-of-birth selector, pre-set cookies)
- Clicks "Add to Account" button
- Verifies claim by checking for success message or "already owned" indicator
- Tracks all results in `data/steam.json`

### Login
- Uses `STEAM_EMAIL` / `STEAM_PASSWORD` environment variables (falls back to `EMAIL` / `PASSWORD`)
- Handles Steam Guard two-factor authentication (5-character code input)
- Supports manual browser login via VNC

### Integration
- `src/config.js`: Added `steam_email`, `steam_password`, `steam_min_rating`, `steam_min_price`
- `Dockerfile`: Added `node steam` to default CMD
- `docker-compose.yml`: Added `STEAM_MIN_RATING` and `STEAM_MIN_PRICE` documentation
- `docker-entrypoint.sh`: Added `node steam` to the script execution chain

---

## Interactive VNC Login Panel

### New file: `interactive-login.js`
A web-based control panel for establishing browser sessions manually. Designed for Docker environments where you need to solve captchas, handle MFA, or complete phone verification through a visible browser.

### Features
- Runs on port 7080 (configurable via `PANEL_PORT`)
- Password protection via `PANEL_PASSWORD` or `VNC_PASSWORD`
- Embeds noVNC viewer showing the Chromium browser on the Xvfb display
- Site buttons for all 4 stores: Prime Gaming, Epic Games, GOG, Steam
- "Login" launches a visible browser navigated to the site's login page
- "I'm Logged In" verifies the session and saves the persistent browser profile
- "Check" verifies an existing session status without opening a browser
- "Check All Sessions" button verifies all sites at once
- "Test Run All Scripts" runs all claiming scripts with live log output
- 4-step progress indicator and contextual status banners

### Docker integration
- `docker-entrypoint.sh`: `LOGIN_MODE=1` launches the interactive panel instead of automated claiming
- `Dockerfile`: Added `ENV PANEL_PORT=7080` and `EXPOSE 7080`
- `docker-compose.yml`: Added port `7080:7080` mapping and `LOGIN_MODE` documentation

### Config
- `src/config.js`: Added `login_mode: process.env.LOGIN_MODE == '1'`

---

## Logging Overhaul

### Shared logging helpers in `src/util.js`
Added `log` object with structured output methods:
- `section(title)` / `sectionEnd()` — Section headers/footers with `─` dividers
- `status(label, value)` — Key-value metadata (2-space indent)
- `info(msg)` — Section-level info with green `✓` (2-space indent)
- `game(name, status)` — Game listing with blue name and arrow (4-space indent)
- `ok(msg)` — Game-level success with green `✓` (4-space indent)
- `skip(name, reason)` — Game-level skip with red `✗`, dim name, yellow reason (4-space indent)
- `warn(msg)` — Game-level warning with yellow `!` (4-space indent)
- `fail(msg)` — Section-level failure with red `✗` (2-space indent)
- `summary(parts)` — Summary line with dim label

### Startup banner in `docker-entrypoint.sh`
- Boxed banner using `═` characters showing version, source URL, branch, and build timestamp
- VNC/noVNC info formatted with consistent indentation

### Full console audit (all 4 scripts)
Converted all raw `console.log`/`console.error`/`console.info` in main flow paths to `log.*` helpers:
- **Login flows**: `log.warn`/`log.info`/`log.status` for sign-in, MFA, captcha, timeout
- **Claim flows**: `log.game`/`log.ok`/`log.fail`/`log.skip` for game processing
- **Redeem flows** (Prime Gaming): `log.ok`/`log.info`/`log.warn` for codes, URLs, store messages
- **DLC flows** (Prime Gaming): `log.info`/`log.status`/`log.game`/`log.warn`/`log.fail` for in-game content

### Noise reduction and debug gating
- `waitUntilStable` timing output → gated behind `DEBUG=1`
- `skipBasedOnTime` timing data → gated behind `DEBUG=1`
- `dismissAgeGate` message → gated behind `DEBUG=1`
- Mature content notices → gated behind `DEBUG=1`
- Bundle-includes parse errors → gated behind `DEBUG=1`
- EULA HTML dumps → gated behind `DEBUG=1`
- Full exception stacks → gated behind `DEBUG=1` (one-line `log.fail()` always shown)
- Raw URL arrays → gated behind `DEBUG=1`

### Consistency
- Em-dash (`—`) used as separator in all `log.warn`/`log.fail`/`log.skip` messages
- Unused `chalk` imports removed from `epic-games.js`, `gog.js`, `steam.js` (chalk used only in `src/util.js` and `prime-gaming.js` for redeem codes)

### Epic Games platform dedup
- Mobile games (Android + iOS) with the same title are deduplicated in output
- Shows unique count with note: `Free games found: 3 (4 incl. platform variants)`
- Per-game suffix when applicable: `(2 platforms)`

---

## Docker / Infrastructure

### Dockerfile
- Added `node steam` to default CMD
- CMD order: `node prime-gaming; node epic-games; node gog; node steam` (Prime Gaming first — most reliable/fastest)
- Added `ENV PANEL_PORT=7080` and `EXPOSE 7080`

### docker-entrypoint.sh
- Added `LOGIN_MODE=1` check for interactive login panel
- Added `-s` (subreaper) flag to all `tini` calls to silence PID 1 warning
- Startup banner redesign with `═` box drawing
- Build metadata display (commit, branch, timestamp)

### docker-compose.yml
- Added port `7080:7080` for interactive login panel
- Added `LOGIN_MODE`, `STEAM_MIN_RATING`, `STEAM_MIN_PRICE` documentation

### GitHub Actions
- Added `.github/workflows/docker-publish.yml` — builds and pushes Docker image to `ghcr.io` on push to `main`

---

## Replit-Specific Files (not for upstream)

These files support running the project in the Replit environment and should not be included in upstream PRs:

- `run.sh` — Launcher script that resolves Nix mesa/libgbm `LD_LIBRARY_PATH` for Chromium
- `scripts/post-merge.sh` — Runs `npm install` and `npx patchright install chromium` after task merges
- `replit.nix` — Nix environment configuration
- `.replit` — Replit project configuration

---

## Summary of All Changed Files

| File | Change Type | Description |
|------|-------------|-------------|
| `steam.js` | **New** | Steam free-to-keep game claimer with SteamDB discovery |
| `interactive-login.js` | **New** | Interactive VNC login panel with 4-site support |
| `prime-gaming.js` | Modified | patchright import, login bug fix, awaited notify(), log.* audit, DLC flow cleanup |
| `epic-games.js` | Modified | patchright import, awaited notify(), log.* audit, platform dedup, removed "in library" notification |
| `gog.js` | Modified | patchright import, awaited notify(), selector fix, log.* audit |
| `src/util.js` | Modified | Removed stealth()/launchChromium(), added `log` helper object |
| `src/config.js` | Modified | Removed AliExpress config, added login_mode, Steam config |
| `Dockerfile` | Modified | patchright, PANEL_PORT, CMD order, added `node steam` |
| `docker-compose.yml` | Modified | Port 7080, LOGIN_MODE, Steam config docs |
| `docker-entrypoint.sh` | Modified | LOGIN_MODE check, tini -s flag, startup banner |
| `package.json` | Modified | patchright dep, docker port 7080 |
| `.github/workflows/docker-publish.yml` | **New** | Auto-build and push Docker image to ghcr.io |
| `aliexpress.js` | **Deleted** | Unused AliExpress script |
| `steam-games.js` | **Deleted** | Unused Steam library scraper |
| `unrealengine.js` | **Deleted** | Unused Unreal Engine script |
| `src/migrate.js` | **Deleted** | One-time migration script |
| `src/version.js` | **Deleted** | Unused version utility |
