# AEM — After Effects Multiplayer

Host and join live real-time ("multiplayer") sessions in After Effects: multiple AE instances on
different machines work on the **same `.aep`** together, and every layer's transform properties
(**position, anchor, scale, rotation, opacity**) sync live. The host's **project and resources are
shared into the room automatically** — clients download them and can open the very same project.

No port-forwarding, no VPN, no hand-copied files: sessions run over the **global internet** through
a public relay at `wss://moonlit.onl/aem/relay/`. (`relay.js` is bundled in the release so you can
also host your own relay.)

## How it works (TL;DR)

- **`AEM.aex`** — a native AEGP plugin that lives inside After Effects. It polls each layer's
  transform streams on every idle tick, sends *small ops* ("set position of layer 3 to [960,540]")
  to the room, and applies remote ops back onto the local project. Because every machine opened the
  **same `.aep`**, comp/layer indices match 1:1, so a remote change replays exactly.
- **CEP panel** — the lobby UI. Shows the room code, hosts/joins/leaves, and runs the file engine:
  it pushes the host's project + resources into the room and downloads them on join.
- **`relay.js`** — a zero-dependency Node room server. Routes ops and stores room files for
  download. Default endpoint is the hosted `wss://moonlit.onl/aem/relay/`.

See **[HOWITWORKS.md](HOWITWORKS.md)** for the full architecture and wire protocol, and
**[HOWTOUSE.md](HOWTOUSE.md)** for setup + usage.

## Features

- Live sync of every layer's position, anchor, scale, rotation and opacity in the active comp.
- Automatic project sharing — the host's `.aep` and every resource are pushed to the room.
- Auto-import — new files imported by anyone are shared to the whole lobby automatically.
- Auto-connect — the plugin remembers your last session and re-hosts/re-joins on load.
- Works worldwide by default, with an optional self-hosted relay for LAN/private use.

## Requirements

- Windows + **After Effects** (plugin compiled for AE 2026; CEP panel works on AE 2024+).
- Internet access (for the default global relay).
- At least two machines to see the sync in action.

## Install (every machine)

1. Run `install.bat` **as Administrator** → copies `AEM.aex` to
   `C:\Program Files\Adobe\Adobe After Effects 2026\Support Files\Plug-ins`.
2. Run `install_extension.bat` (double-click) → installs the panel for your user and enables
   unsigned CSXS extensions.
3. Restart After Effects.

## Quick start

1. Host opens their project, opens the panel (**Window > Extensions > AEM Multiplayer**) and clicks
   **Host Session**. The panel shows the room code and starts sharing the project.
2. Clients enter the host's **Room code** and click **Join Session**. The project downloads,
   opens itself, and resources import automatically.
3. Edit on any machine — transforms sync in real time to everyone.
4. Import a new file on any machine — everyone in the room gets it.
5. Click **Leave** when done.

## Relay

- **Default:** `wss://moonlit.onl/aem/relay/` — the plugin connects here unless told otherwise.
- **Override per machine:** change the **Relay** box in the panel, or write a URL into
  `%LOCALAPPDATA%\AEM\relay.txt` (delete the file or write `default` to return to the global relay).
- **Self-host (optional):** run `node relay.js` (port 5558 by default; rooms under
  `AEM_ROOMS_DIR/<code>/`), point every machine's Relay box at `ws://<server-ip>:5558`, and open
  inbound TCP 5558 on the server.

## Notes & limitations (MVP)

- Syncs the **active comp** in the foreground (one comp at a time).
- **Last-write-wins** — whoever edits last is the source of truth; no conflict resolution yet.
- Shared files are deduped by path — a resource changed only *on disk* won't re-share until it's
  re-imported under a new path.

## Building from source

- **Plugin (C++):** CMake project under `aegp/plugin` (AE SDK 25.6). `build.bat` compiles and
  drops a versioned `.aex` into `output\`.
- **Panel (HTML/JS):** `cep/AEM`, no build step.
- **Relay:** `relay/relay.js`, zero npm dependencies.
- Run `sync.bat` to copy the built plugin, panel, relay and docs into `output\` — the distributable
  folder.

## License

No license specified. See the repository owner before redistributing.
