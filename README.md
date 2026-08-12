# AEM — After Effects Multiplayer

Real-time multiplayer collaboration for After Effects 2026 — multiple editors working in the same project at the same time, with live sync across transforms, keyframes, comp settings, and effect parameters.

> ⚠️ **Status: early build.** The UXP panel client is implemented; the relay server (Node/WebSocket) is a separate project this panel talks to and still needs building. See [Architecture](#architecture) and [Known limitations](#known-limitations).

---

## What this is

After Effects has no built-in way to collaborate live — no shared sessions, no event API to hook into for detecting edits. AEM builds that layer as a UXP panel:

- **Live property sync** — transforms, keyframes, comp settings (duration/frame rate/width/height), and effect parameters propagate to everyone else's AE instance, via a generic property-tree walker rather than per-effect hand coding.
- **Layer claims (soft locking)** — a layer is claimed the moment someone actively edits it (not on mere selection). Claims cascade through parent/null rigs to their children, and auto-release after 5 seconds of inactivity — which also covers a claimant disconnecting mid-edit.
- **Identity system** — persistent username + color per user, server-confirmed collision handling: duplicate names get a `#XXXX` tag, duplicate colors get silently reassigned, red is reserved exclusively for that reassignment.
- **Collaboration UI** — layer list with claim badges and color-coded ownership outlines, a presence strip showing who's connected and which comp they're in, and toast feedback when an edit gets blocked by someone else's claim.

## Why it's hard

After Effects' scripting API is **eventless** — there's no `onPropertyChanged` callback. Every part of this system is built around **polling and diffing** project state (every 200ms by default) rather than subscribing to changes. That single constraint shapes nearly every architectural decision here.

## Architecture

```
┌─────────────┐   WebSocket    ┌────────────────────┐   WebSocket   ┌─────────────┐
│ UXP Panel A │ ◄─────────────►│  Relay Server      │◄─────────────►│ UXP Panel B │
│ (in AE)     │                │ (Node + WebSocket, │               │ (in AE)     │
│             │                │  moonlit.onl)      │               │             │
└─────────────┘                └────────────────────┘               └─────────────┘
      │                                                                    │
      ▼                                                                    ▼
 Local AE project                                                   Local AE project
 (applied via UXP/                                                  (applied via UXP/
  ExtendScript bridge)                                              ExtendScript bridge)
```

Each editor runs the project locally. The panel polls `app.project`'s active comp, diffs it against the last snapshot, and sends only what changed to the relay, which rebroadcasts it to everyone else in the room. The relay never touches AE directly — it's a message router plus identity/collision arbitration, not a project-state owner.

## Project structure

```
AEM/
├── manifest.json      UXP manifest — panel entrypoint, moonlit.onl network permissions
├── index.html          panel shell: identity picker, session controls, layer list, log
├── css/panel.css
└── js/
    ├── identity.js      username/color persistence + server-confirmed collision handling
    ├── relay.js         WebSocket client — host/join, sync ops, message contract
    ├── walk.js          generic property-tree walker (transform + effects, by matchName)
    ├── diff.js           snapshot diffing — static values, keyframes, comp settings
    ├── apply.js          writes remote ops back into AE, feedback-loop guard
    ├── claims.js          per-layer soft locking, parent-chain cascade, idle release
    ├── ui.js               layer badges/outlines, presence strip, toast, identity card
    └── main.js             wires everything together, poll loop
```

## Message contract (panel ↔ relay)

```
-> host   {t:'host', code, name, color}
-> join   {t:'join', code, name, color}
<- ok     {t:'ok', code, you, tag, color, peers}     // server-confirmed identity
<- peer   {t:'peer', from, e:{...}}                   // relayed from another client
-> sync   {t:'sync', e:{act:'set', path, static, keys, layer}}
-> sync   {t:'sync', e:{act:'claim', layer, chain, ts}}
-> sync   {t:'sync', e:{act:'release', chain}}
-> sync   {t:'sync', e:{act:'presence', name, color, comp}}
<- err    {t:'err', e}
```

The relay server implementing this contract (session creation, room routing, username/color collision arbitration) is not part of this repo yet — it's the next piece to build.

## Known limitations

- **No CRDT.** Conflict resolution is currently last-write-wins at the property level, backstopped by the claim system rather than a true merge algorithm (e.g. Yjs). This covers the common case well since claims prevent most concurrent edits to the same layer, but doesn't guarantee a deterministic merge if a claim is somehow bypassed.
- **No structural sync.** Adding/removing a layer or effect mid-session isn't synced yet — only changes to properties that already exist on both sides.
- **AE's undo stack is local-only.** A remote change landing mid-session isn't reflected in your local undo history.
- **Keyframe tracks are capped** at 64 keys per property for diffing safety; longer tracks are truncated in the synced snapshot.

## Pricing

AEM is a paid extension — **€15 / $16**, one-time. Purchase includes the panel client and access to the hosted relay (moonlit.onl) needed to actually create/join rooms; the client isn't useful standalone without a relay to talk to.

> **Current status:** dev builds (including this one) are free to use for now — license key + HWID-lock enforcement hasn't been built yet. This is a pre-launch state, not the final paid product. Once licensing ships, dev builds will stop being distributed and a purchased key will be required to run AEM.

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
