# AEM — After Effects Multiplayer (Plugin + Extension)

Host and join live real-time ("multiplayer") sessions in After Effects: multiple AE instances on
different machines work on the **same .aep** together, and **every layer's transform properties**
(position, anchor, scale, rotation, opacity) sync live. The host's **project and resources are
shared into the room automatically** — clients download them and can open the very same project.

It works over the **global internet** via the relay at `wss://moonlit.onl/aem/relay/` — no local
server or port-forwarding needed. (`relay.js` is also bundled so you *can* host your own.)

## What's in this folder

| File | Purpose |
|------|---------|
| `AEM.aex` | The native AEGP plugin ("hands"). Lives in the AE Plug-ins folder. |
| `cep\AEM\` | The CEP panel ("brain"). Lobby UI + file sharing + orders. |
| `install_extension.bat` | Installs the panel for your current Windows user (no admin needed). |
| `install.bat` | Copies `AEM.aex` into AE 2026's Plug-ins folder (needs admin). |
| `relay\relay.js` | The relay server (runs on `moonlit.onl` — optional to self-host). |
| `HOWTOUSE.md` | This file. |
| `HOWITWORKS.md` | How the whole thing works internally. |

## Requirements

- Two or more machines, each with **After Effects** (plugin is compiled for AE 2026; panel works
  on AE 2024+).
- Internet access (default relay: `wss://moonlit.onl/aem/relay/`).
- (No need to copy files by hand — the host's project + resources are shared through the panel.)

## Setup (do once, on every machine)

1. Run `install.bat` **as Administrator** → copies `AEM.aex` to
   `C:\Program Files\Adobe\Adobe After Effects 2026\Support Files\Plug-ins`.
2. Run `install_extension.bat` (double-click) → copies the panel to
   `%APPDATA%\Adobe\CEP\extensions\AEM` and enables unsigned-extensions for CSXS.
3. Restart After Effects.

## How to use

1. **Host**: open your project in AE, then run the panel (**Window > Extensions > AEM
   Multiplayer**) and click **Host Session** (or do nothing — the plugin auto-hosts if no session
   is saved). The panel shows your room code and starts sharing your `.aep` + every resource.
2. **Clients**: put the host's code in the **Room code** box, click **Join Session**. Your panel
   downloads the host's files to the exact same paths, **opens the .aep**, and imports the
   resources. (Leave the **Relay** box as the default — it points at the global relay.)
3. Edit layers on any machine. **Position, Anchor, Scale, Rotation & Opacity of every layer**
   in the active comp sync to everyone else in real time.
4. **Anyone who imports a new file** (footage, audio, image) into the project — the new resource is
   automatically shared to the whole room.
5. Done: click **Leave**.

### Auto-connect

The plugin remembers your last session (`%LOCALAPPDATA%\AEM\state.txt`) and auto-connects on load —
hosts the same code again if you were host, joins again if you were a client. The panel just shows
what the plugin is doing.

## Notes & limitations (MVP)

- Syncs the **active comp** in the foreground (one comp at a time).
- **Last-write-wins**: whoever edits last is the source of truth.
- No collision handling if two users edit at the same instant.
- Keep internet access to the relay; otherwise edits won't propagate.
- Shared files are deduped by path — a resource that only *changes on disk* won't re-share until
  it's imported again under a new path.

## Relay (global by default, optional self-host)

- **Default**: `wss://moonlit.onl/aem/relay/`. The plugin connects there unless told otherwise.
- **Override**: every machine's **Relay** box in the panel overrides the endpoint for that machine
  (e.g. `wss://my-relay.example/aem/relay/` or a LAN `ws://192.168.1.50:5558`). Leave it as the
  moonlit.onl default for worldwide use.
- The endpoint is stored in `%LOCALAPPDATA%\AEM\relay.txt` (one line). Delete that file or write
  `default` to return to the built-in global relay.

**Self-hosting (optional):** run `node relay.js` (zero npm deps, port 5558 by default), put
`ws://<server-ip>:5558` in every machine's Relay box, and open inbound TCP 5558 on the server.
Room files are stored on the server under `AEM_ROOMS_DIR/<code>/` (default: `aem_rooms/`).