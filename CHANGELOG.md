# Changelog

All notable changes to this project are recorded here. Format loosely follows
[Keep a Changelog](https://keepachangelog.com/); versions by plugin/panel version.

## [Unreleased]

### Added
- **Session service (v2):** `GET https://moonlit.onl/aem/session` issues a room id — a custom
  **room name** (e.g. `MYLOBBY`) for private rooms, or an auto-generated `AEM####`. Names can't be
  reused while the room is live (`{ok:false,"err":"taken"}`).
- **Path-addressed rooms (v2):** the room is the URL path (`wss://moonlit.onl/aem/relay/<SESSION>`).
  The panel accepts a bare name/code **or** a full link; both resolve to the same room.
- `relay.js` still serves bare `/aem/relay/` (fallback to the `code` in the first frame), so old
  clients keep working.

### Changed
- **Host** in the panel now calls the session service first (typed name or empty → auto id), sets
  the code box, and points the plugin at the full session URL.
- **Join** accepts a bare name/code or a full `wss://…/aem/relay/<SESSION>` link.
- `updatePluginRelay()` writes the full session URL to `relay.txt` (or `default` for the global
  endpoint).
- HOWTOUSE / HOWITWORKS updated for the session model.

### Fixed
- Host/Join no longer fail out of the box. The panel now defaults to the **global relay**
  (`wss://moonlit.onl/aem/relay/`) instead of assuming a local relay, so sessions work between
  machines on completely different networks.
- The plugin is only pointed at an in-extension/local relay when the user explicitly types a
  `ws://127.0.0.1`, `ws://localhost` or `ws://0.0.0.0` address into the Relay box.

### Added
- `sync.bat` — one-command repopulation of the `output\` distributable (plugin, panel, relay,
  docs) from the sources.

## [1.0.0] — architecture v4: plugin = hands, extension = brain + file engine

### Added
- Native AEGP plugin (`AEM.aex`) that reads every layer's transform streams each idle tick and
  applies incoming remote ops back onto the project via AEGP suites.
- CEP panel ("brain") with lobby UI: room code display, Host / Join / Leave controls, live layer
  list, and status.
- File engine: the host's `.aep` and every project resource are pushed into the room
  (hash-addressed) and downloaded to matching paths on join, then opened/imported automatically.
- Auto-import: new files imported by any user are shared to the whole room.
- Auto-connect: plugin persists the last session (`%LOCALAPPDATA%\AEM\state.txt`) and re-hosts or
  re-joins automatically on load.
- Global-connectivity: a hosted relay at `wss://moonlit.onl/aem/relay/` with no port-forwarding.
- Optional self-hosted relay (`relay.js`, zero npm deps) with on-disk room storage.

### Changed
- Wire protocol: JSON text frames over WebSocket (`host` / `join` / `sync` / `leave` / `ok` /
  `peer` / `err`); ops like `{"act":"set","layer":3,"prop":"position","v":[...]}`.
- Layer indices 1-based on the wire, 0-based internally in AEGP.

### Fixed
- Relay endpoint now overridable per machine via `%LOCALAPPDATA%\AEM\relay.txt` or the panel's
  Relay box (fallback to the compiled global endpoint).

## [0.x] — earlier prototypes

### Added
- Minimal two-instance Position + Opacity sync for layer 1 of the active comp over a hand-rolled
  WebSocket relay (MVP proof of concept).
- AEGP plugin skeleton (menu commands Host / Join / Leave, idle-hook diff loop, diff cache to
  avoid replay spam on load).

<!--
Template for future entries:

## [X.Y.Z] — date

### Added
### Changed
### Fixed
### Removed
-->
