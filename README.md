# AEM - After Effects Multiplayer

Real-time multiplayer collaboration for After Effects: multiple editors working in the same project at the same time, with live sync across transforms, keyframes, comp settings, effect parameters, layer structure, and project assets.

> ⚠️ **Status: early build.** The panel + host script are implemented and talk to a hosted relay (moonlit.onl) for room routing and file transfer. This is a debug/unsigned build, see [Installation](#installation).

---

## What this is

After Effects has no built-in way to collaborate live: no shared sessions, no event API to hook into for detecting edits. AEM builds that layer as a **CEP extension** (panel + ExtendScript host):

- **Live property sync**: transforms, keyframes, comp settings (duration/frame rate/width/height), and effect parameters propagate to everyone else's AE instance, via a generic property-tree walker rather than per-effect hand coding.
- **Structural sync**: layers can be added, removed, and reordered mid-session and stay in sync, via stable per-layer IDs that survive index shifts (not raw layer-index tracking).
- **Project & asset sharing**: the host's `.aep` and every footage/audio/image file it references get pushed to joining peers in chunks (100 MB/file cap) and imported automatically; anyone who imports new footage mid-session shares it too, so the whole room converges on the same assets.
- **Layer claims (soft locking)**: a layer is claimed the moment someone actively edits it, not on mere selection. Claims cascade through parent/null rigs to their children, and auto-release after 5 seconds of inactivity, which also covers a claimant disconnecting mid-edit.
- **Identity system**: persistent username + color per user, server-confirmed collision handling. Duplicate names get a `#XXXX` tag, duplicate colors get silently reassigned, red is reserved exclusively for that reassignment.
- **Collaboration UI**: layer list with claim badges and color-coded ownership outlines, a presence strip, transfer progress for shared files, and toast feedback when an edit gets blocked by someone else's claim.

## Why it's hard

After Effects' scripting API is **eventless**. There's no `onPropertyChanged` callback. Every part of this system is built around **polling and diffing** project state rather than subscribing to changes. That single constraint shapes nearly every architectural decision here.

## Why CEP, not UXP

Earlier builds targeted UXP, but After Effects doesn't support UXP panel plugins yet (that's still an Adobe roadmap item), so this build uses **CEP** (HTML panel + ExtendScript), the framework every commercial AE panel runs on today.

## Architecture

```
┌───────────────────────┐   WebSocket    ┌──────────────────┐   WebSocket   ┌────────────────────────┐
│ CEP Panel A           │◄─────────────► │  Relay Server    │◄─────────────►│ CEP Panel B            │
│ (client/ + evalScript)│                │ (moonlit.onl,    │               │ (client/ + evalScript) │
│         │             │                │  rooms + file    │               │         │              │
│         ▼             │                │  relay)          │               │         ▼              │
│ host/AEM.jsx          │                └──────────────────┘               │ host/AEM.jsx           │
│ (ExtendScript, in AE) │                                                   │ (ExtendScript, in AE)  │
└───────────────────────┘                                                   └────────────────────────┘
```

The panel (`client/`) polls and diffs project state, then talks to AE through `bridge.js`'s `evalScript` wrapper into `host/AEM.jsx`, the ExtendScript backend that actually reads/writes the AE DOM (snapshotting, applying ops, file I/O, project open/save). The relay never touches AE directly; it routes sync messages and shuttles file chunks between peers.

## Installation

See `HOW-TO-INSTALL.md` for full steps. Short version:

1. Extract the zip, keeping the folder named `com.moonlit.aemultiplayer`.
2. Run `install.bat`. This enables CEP debug mode in the registry and copies the extension into `%APPDATA%\Adobe\CEP\extensions`.
3. Restart After Effects, then open **Window → Extensions → AEM**.

This is currently a debug/unsigned build. Distribution as a signed `.zxp` (so end users don't need the registry debug flags) is planned for the paid release.

## Message contract (panel ↔ relay)

```
-> host      {t:'host', code, name, color}
-> join      {t:'join', code, name, color}
<- ok        {t:'ok', code, you, tag, color, peers}          // server-confirmed identity
<- peer      {t:'peer', from, e:{...}}                        // relayed sync event
-> sync      {t:'sync', e:{act:'set', id, path, val, keys}}
-> sync      {t:'sync', e:{act:'addLayer'|'removeLayer'|'moveLayer'|'renameLayer', ...}}
-> sync      {t:'sync', e:{act:'claim', layer, chain, ts}} / {act:'release', chain}
-> sync      {t:'sync', e:{act:'presence', name, color, comp}}
-> file      fileBegin / fileGet  (project + asset transfer, chunked, 100 MB/file cap)
<- file      {op:'added'|'data', name, path, size, data}
<- err       {t:'err', e}
```

## Known limitations

- **No CRDT.** Conflict resolution is last-write-wins at the property level, backstopped by the claim system rather than a true merge algorithm (e.g. Yjs). Claims prevent most concurrent edits to the same layer, but this doesn't guarantee a deterministic merge if a claim is somehow bypassed.
- **Layer identity matching** for unnamed null/solid layers can miss a reorder when two same-type, differently-unnamed layers swap position, noted directly in `ids.js`.
- **AE's undo stack is local-only.** A remote change landing mid-session isn't reflected in your local undo history.
- **Keyframe tracks are capped** at 64 keys per property for diffing safety; longer tracks are truncated in the synced snapshot.
- **File transfers are capped** at 100 MB per file.

## Pricing

AEM is a paid extension: **€29.99 / $34.60**, one-time. Purchase includes the panel client and access to the hosted relay (moonlit.onl) needed to create/join rooms and share project files. The client isn't useful standalone without a relay to talk to.

> **Current status:** dev builds (including this one) are free to use for now. License key + HWID-lock enforcement hasn't been built yet. This is a pre-launch state, not the final paid product. Once licensing ships, dev builds will stop being distributed and a purchased key will be required to run AEM.

## License

© melthedev. All rights reserved.

This code is proprietary and is **not** open source. A paid purchase grants the buyer a personal, non-transferable license to install and use AEM for their own After Effects work. It does **not** grant the right to:

- redistribute, resell, or share the extension or its source with others
- modify and redistribute a derivative version
- reverse-engineer, decompile, or extract the relay protocol for use outside AEM
- use it for commercial resale (e.g. bundling it into another paid product)

No license is granted to anyone who has not purchased AEM. All rights not explicitly granted above are reserved.

---

A project by **melthedev**, founder of the **Moonlit** community.
