# AEM — How It Works

AEM turns After Effects into a multiplayer editor: two or more AE instances stay in sync over the
internet, sharing live edits on the **same project file** — and the actual project files (the
`.aep` plus every resource) are shared into the room automatically.

The default relay is **global** (`wss://moonlit.onl/aem/relay/`), so machines anywhere in the
world can join the same room code. The bundled `relay.js` is also runnable as your own server;
the panel's Relay box overrides the endpoint per machine.

## Architecture (v4: plugin = hands, extension = brain + file engine)

```
  AE instance A (host)                                  AE instance B (client)
  ┌─────────────────────────────────────────────┐        ┌─────────────────────────────────────────┐
  │ AEGP plugin AEM.aex ──ws──┐                 │        │                    ┌──ws──► AEGP plugin │
  │ (watch/write transforms)  │      wss/ws     │        │     wss/ws         │  AEM.aex           │
  │  ▲ state.json ▼ orders    │◄────────────────┼────────┼───────────────────►│ (apply remote ops) │
  │  ▲                        │    relay        │        │    to host's       │                    │
  │  │ file bridge            │ INSIDE AE       │        │    in-AE relay     │                    │
  │  CEP panel                │  (js/relay.js)  │        │    CEP panel       │                    │
  │  (hosts relay on 0.0.0.0) │  :5558 ◄────────┼────────┼──────────          │                    │
  │  └── 0.0.0.0:5558         ┘                 │        │  (file socket →    │                    │
  └─────────────────────────────────────────────┘        │   ws://<host>:5558)│                    │
                                                         └────────────────────┴────────────────────┘
```

Three pieces:

| Piece | What it does |
|-------|--------------|
| **`AEM.aex`** (AEGP plugin) | The **hands**. Runs natively inside AE: polls every layer's transform streams each idle tick, applies incoming remote ops back onto the project via AEGP suites, and owns the WebSocket connection to the relay (with auto-connect). |
| **CEP panel** (brain + file engine) | Runs in AE's embedded Chromium. Reads the plugin's `state.json` and renders the lobby. Sends orders via `orders.json`. Writes the relay endpoint to `%LOCALAPPDATA%\AEM\relay.txt` before Host/Join. Owns a second, **file** WebSocket (`link`) that pushes the open project + its resources to the room and downloads the host's files on join. |
| **`relay.js`** (Node, zero deps) | The room server: routes ops, stores room files on disk (hash-addressed, under `AEM_ROOMS_DIR/<code>`). Runs on the **global** relay host (`moonlit.onl/aem`); also runnable standalone (`node relay.js`) for self-hosting. |

Two independent WebSocket channels share the room managed by the relay:

| Channel | Owned by | Does what |
|---------|----------|-----------|
| **ops** (transform sync) | plugin socket (`host`/`join` peer) | streams `position/anchor/scale/rotation/opacity` edits in real time |
| **file** (project sharing) | panel socket (`link`, not counted as a peer) | pushes the host's `.aep` + every resource; clients download + restore them |

## Relay endpoint selection

The plugin used to hardcode `wss://moonlit.onl/aem/relay/`. Now every `host`/`join` reads
`%LOCALAPPDATA%\AEM\relay.txt` first (`Entry.cpp::relay_endpoint`):

- One line like `ws://127.0.0.1:5558` or `wss://moonlit.onl/aem/relay/` → parsed into
  host/scheme/path/port for the WebSocket client.
- Missing, empty, or `default` → falls back to the compiled default
  (`AEM_RELAY_DEFAULT_URL/Scheme/Path/Port` in `AEM.h`).
- The CEP panel writes this file before Host/Join (`aemSetRelay(url)` in `host/script.jsx`), so
  the host points its plugin at `ws://127.0.0.1:5558` where its own in-panel relay listens, and
  clients point theirs at `ws://<host-ip>:5558`.

## The two file-pair bridges

The AE-side pieces talk through files in `%LOCALAPPDATA%\AEM\`:

- `orders.json` — panel → plugin. A **one-shot order** the plugin reads once on an idle tick and
  deletes: `{"cmd":"host"}`, `{"cmd":"join","code":"AEM1000"}`, `{"cmd":"leave"}`.
- `state.json` — plugin → panel. Written ~every 128 ms: `{connected, role, code, peers, files,
  layers:[...]}`, where `layers` holds each layer's `position/anchor/scale/rotation/opacity`
  (1-based `layer` numbers) and `files` mirrors the room's file manifest for the lobby.
- `relay.txt` — panel → plugin. One line describing the relay endpoint (see above).
- `state.txt` / `room.txt` — plugin persistence for auto-connect and the join code.

## The sync trick: same file, aligned indices

Because every instance opened the same `.aep`, "layer 3 of comp X" means the same thing on every
machine. So the live path ships **small ops** ("set position of layer 1 to
[960,540]") and the plugin replays them against the local identical project via AEGP streams. No
merge, no file lock — just matching indices + per-property writes. Getting everyone onto that same
`.aep` in the first place is the **file channel's** job (host pushes, clients pull).

## What the plugin does each idle tick (~16 ms)

`Entry.cpp::IdleHook`:

1. On the first tick only, **auto-connects**: resumes the last saved session from
   `state.txt` (`host <code>` → hosts again, `client <code>` → joins; nothing saved → auto-hosts).
   Networking lives in `net\ws.h`/`net\ws.cpp` (aem::Wsc, blocking connect outside the AE thread).
2. **Processes orders** (`aem::process_orders`): reads + deletes `orders.json`, runs the command
   (host/join/leave).
3. **Pumps the socket** and, if connected, **diffs the project** (`aem::sync_project_change` in
   `AEM_Sync.cpp`): for every layer of the active comp it reads Position/Anchor/Scale/Rotation/
   Opacity via `AEGP_GetLayerStreamValue` and compares to a cached snapshot. Any change is sent as
   `{"t":"sync","e":{"act":"set","layer":N,"prop":...,"v":[...]}}` (wire layers are 1-based).
4. Every ~8 ticks **writes state.json** (`aem::write_state`) so the panel can render the lobby live.

Phase-1 after connecting is a **reseed** (`reseed=true`): the first pass only records the baseline
into the cache and sends nothing, so everyone starts from the same picture before edits flow.

## What the plugin does with remote ops

Incoming `{"t":"peer","e":{op}}` from the relay → `aem::on_net_message` → `aem::apply_local_op(e)`:

- `prop_stream()` maps wire names → AEGP streams: `position`→POSITION, `anchor`→ANCHORPOINT,
  `scale`→SCALE, `rotation`→ROTATION, `opacity`→OPACITY.
- `write_stream()` grabs a fresh layer stream ref (`AEGP_GetNewLayerStream`), sets the value
  (`AEGP_SetStreamValue`), disposes it.
- The write updates the local cache so the change isn't echoed back out to the room.

## What the panel does

`%APPDATA%\Adobe\CEP\extensions\AEM\js\index.js` (CEP Chromium):

- Polls the bridge every ~150 ms: `evalScript('aemReadState()')` → `host/script.jsx` reads
  `state.json`, returns it as a JSON string.
- **Relay**: with `--enable-nodejs`, `require` (`window.cep_node.require`) loads
  `js/relay.js`; `startRelay()` binds `0.0.0.0:5558` (rooms in `os.tmpdir()/aem_rooms`). Hosting
  calls this and reports the host's LAN IP so clients know the URL to enter.
- Renders the lobby: role, room code, peer count, room file manifest, per-layer transform readout.
- Buttons write orders via `aemSendOrder({...})` → `orders.json`, and set the relay endpoint via
  `aemSetRelay(url)` → `relay.txt`. Host/Join/Leave are just file writes; the plugin networks.
- When connected, opens its **file WebSocket** as a `link` and on the `ok` manifest:
  - **host** → `shareProject()`: reads `aemProjectFiles()` (the `.aep` + every footage/audio/image
    path the open project references) and streams each file to the relay in 1MB base64 chunks.
  - **client** → `startPull()`: `get`s every file and writes it to the **host's exact absolute
    path** (`aemWritePath`), then `aemOpenProject` for the `.aep` and `aemImportFile` per resource.
- `watchProject()` every ~2.5 s diffs the project's file list against a baseline, so a resource
  **anyone newly imports** is pushed to the whole room (clients' pre-join files stay private).

## Session / room lifecycle

1. Plugin auto-hosts on load (or the panel's **Host Session** order does the same). The panel's
   Host click first starts the in-AE relay and points the plugin at it via `relay.txt`.
2. Client's **Join Session** (with the host's relay URL + code) → plugin connects and sends
   `{"t":"join","code":...,"name":...}`.
3. Relay replies `{"t":"ok","you":"host|client|link","code":…,"peers":n,"files":[...]}` → plugin
   records `connected` + `peer_count`, which flows into `state.json` for the panel.
4. **Leave** → plugin sends `{"t":"leave"}`; the room vanishes when the last peer leaves.

## The wire protocol

Plain UTF-8 JSON text frames. Two channels share the same relay endpoint:

| Direction | Message | Meaning |
|-----------|---------|---------|
| plugin → relay | `{"t":"host","code","name"}` | claim a room code (host) |
| plugin → relay | `{"t":"join","code","name"}` | enter an existing room (client) |
| plugin → relay | `{"t":"sync","e":{op}}` | broadcast an edit op to the room |
| plugin → relay | `{"t":"leave"}` | leave / disconnect |
| panel → relay | `{"t":"link","code"}` | attach the file channel (not counted as a peer) |
| panel → relay | `{"t":"file","op":"begin\|chunk\|end","name","path","data"}` | push a file in base64 chunks |
| panel → relay | `{"t":"file","op":"get","name","path"}` | pull a stored room file |
| relay → plugin | `{"t":"ok","you","code","peers","files":[...]}` | joined; `you=host\|client\|link` + room manifest |
| relay → plugin | `{"t":"peer","from","e":{op}}` | a remote edit op forwarded to you |
| relay → plugin | `{"t":"file","op":"added","name","path","size"}` | a peer just finished sharing a file |
| relay → plugin | `{"t":"file","op":"data","name","size","data"}` | requested file bytes (for `get`) |
| relay → plugin | `{"t":"err","e":"reason"}` | e.g. `code-taken`, `no-host` |

Op body: `{"act":"set","layer":3,"prop":"position","v":[960,540]}` (or a scalar `v` for
rotation/opacity). `act` currently supports `set`; `add-kf` (keyframes) and `effect` are reserved.

## Relay internals (`relay.js`)

- Zero npm dependencies; hand-rolled RFC 6455 handshake + frame encode/decode (mask/unmask,
  7/16/64-bit lengths, fragmented frames dropped).
- **Two launch modes**: `node relay.js` standalone, or the panel `require`s it and calls
  `startRelay({port, host, roomsDir})` (module export; the CLI guard only fires when run as main).
- Rooms: `Map<code, {host, clients, linked, files}>` — one host, any number of client (peer) conns,
  any number of `linked` conns (the panels' file channels), all in memory.
- `broadcast` fans a `sync` or a `file/added` notification out to **everyone except the sender**
  (host + clients + linked included).
- Start/stop messages (`host`/`join`/`link`) reply with an `ok` carrying `peers` + the current
  `files` manifest, so every panel reconciles its local state on (re)connect.
- **File store**: each room lives on disk under `AEM_ROOMS_DIR/<code>/` as hash-addressed blobs +
  an `index.json` mapping hash → `{name, path, size}`. `get` streams a file back in ~4MB base64
  frames.
- Keepalive: pings every socket every 15 s; dead conns are dropped and rooms cleaned up.

## Why the mix (native + extension)

- The AEGP plugin gives us **any layer, any transform stream**, at full AE-native speed, on a real
  socket — and survives AE restarts (auto-connect). ExtendScript alone was too slow/flaky for
  real-time per-layer polling.
- The CEP panel gives us a proper UI without C++ kitchen-sinkery, **and a real Node runtime**, so
  the WebSocket relay itself runs inside AE — no separate server process to babysit.
- The relay stays a dumb pipe that both channels share.

## Known limits (MVP)

- The relay runs in the **host's AE process**; the host must stay running and reachable.
- Syncs the **active comp** only; property streams are Position/Anchor/Scale/Rotation/Opacity.
- **Last-write-wins**: simultaneous edits can interleave; no CRDT/OT.
- No keyframe animation sync (`add-kf` reserved for later).
- Files shared once are not re-pushed when their contents change on disk (dedupe is by path); the
  watch only detects **new** paths (imports).
- On a dropped socket there is no reconnect loop — re-Host/Join (the plugin does auto-connect on load).
- The file bridge is per-user (`%LOCALAPPDATA%AEM`); two AE instances on one Windows account
  would share the same state/order files.
