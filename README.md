# AE Multiplayer

Real-time multiplayer collaboration for After Effects 2026 — multiple editors working in the same project at the same time, with live sync across transforms, keyframes, comp settings, and effect parameters.

> ⚠️ **Status: Planning / early build.** This is an ambitious systems project built around AE's lack of native change-event hooks. See [Architecture](#architecture) and [Risks](#risks) before diving in.

---

## What this is

AE has no built-in way to collaborate live — no shared sessions, no event API to hook into for detecting edits. This project builds that layer from scratch:

- **Full live sync** — any user can edit any property (transforms, keyframes, comp settings, effect parameters) and it propagates to everyone else's AE instance.
- **Layer claims (soft locking)** — when a user actively edits a layer, it's claimed: other users see who owns it and can't edit it until they're done (5s idle timeout releases it automatically). Claims cascade through parent/null rigs.
- **Presence & identity** — persistent username + color per user, collision-safe (duplicate usernames get tagged, duplicate colors get silently reassigned).
- **Collaboration UI** — layer list overlay with claim badges, color-coded ownership outlines, session status bar, and connection health, all inside a UXP panel.

## Why it's hard

After Effects' scripting API is **eventless** — there's no `onPropertyChanged` callback. Every part of this system is built around **polling and diffing** project state rather than subscribing to changes. That single constraint shapes nearly every architectural decision here.

## Architecture

```
┌─────────────┐   WebSocket    ┌──────────────────┐   WebSocket   ┌─────────────┐
│ UXP Panel A │ ◄─────────────►│  Relay Server    │◄─────────────►│ UXP Panel B │
│ (in AE)     │                │ (Node + Yjs/     │               │ (in AE)     │
│             │                │  y-websocket)    │               │             │
└─────────────┘                └──────────────────┘               └─────────────┘
      │                                                                 │
      ▼                                                                 ▼
 Local AE project                                                Local AE project
 (applied via UXP/ExtendScript bridge)                           (applied via UXP/ExtendScript bridge)
                                           
```

Each editor runs the project locally. UXP panels poll AE's project state, diff it against the last known snapshot, and push changes through a CRDT (Yjs) via a relay server that merges and rebroadcasts updates. The relay never touches AE directly — it's a dumb merge point.

## Core components

| Component | Purpose |
|---|---|
| **Relay server** | Node + Yjs/y-websocket. Holds the canonical CRDT doc per session, merges and rebroadcasts updates, tracks presence. |
| **Poll & diff engine** | Walks `app.project` every 100–250ms, diffs against last snapshot, emits only changed property paths. |
| **CRDT mapping layer** | Maps AE property paths to Yjs shared types for deterministic concurrent-edit merging. |
| **Apply layer** | Writes incoming remote changes back into the local AE project via the UXP/ExtendScript bridge. |
| **Presence/awareness** | Ephemeral state — who's connected, active comp, selection — via Yjs Awareness. |
| **Layer claims** | Soft per-layer locking triggered by active edits (not selection), cascading through parent chains, released after 5s idle. |
| **Collaboration UI** | Layer list overlay, color-coded claim outlines, session status bar, lock toasts, comp presence strip. |
| **User identity system** | Username + color, persisted locally, collision-safe (`#XXXX` tag for duplicate names, silent reassignment for duplicate colors — red reserved as the fallback). |

## Tech stack

- **Host:** After Effects 2026 (UXP, not CEP)
- **Panel:** UXP + ExtendScript bridge for AE scripting calls
- **Sync:** [Yjs](https://github.com/yjs/yjs) (CRDT) + [y-websocket](https://github.com/yjs/y-websocket)
- **Server:** Node.js

## Build roadmap

1. Relay server skeleton (session/room handling)
2. UXP panel skeleton (round-trip connectivity)
3. Poll + diff engine — transform properties only
4. Apply layer — remote diffs written back into AE
5. Expand snapshot scope — keyframes, comp settings, markers
6. Generic property-tree walker — full effect/expression coverage
7. Presence/awareness layer
8. Layer claims — trigger, cascade, idle release
9. Conflict handling hardening
10. Feedback-loop guards + performance pass
11. Collaboration UI
12. User identity system

## Open decisions

- Conflict policy for two users editing the exact same property at once
- Poll interval tradeoff (responsiveness vs. AE UI thread load)
- Session/auth model for the relay server
- Whether relay state needs persistence beyond the local `.aep` file
- v1 effect coverage: AE-native only, or third-party plugins too

## Risks

- **UXP/ExtendScript bridge limitations** — not every AE scripting operation may be exposed identically across UXP vs. legacy ExtendScript; needs early verification.
- **Poll performance** — full-tree polling at 100–250ms could be expensive on large comps; may need scoped/dirty-region polling.
- **Merge correctness** — CRDTs handle structural conflicts well, but AE-specific semantics (interpolation types, expression side effects) may need custom merge logic beyond generic primitives.
- **Local undo stack** — AE's undo is local-only; remote-originated changes landing mid-session can conflict with a user's Ctrl+Z expectations. Flagged as a known v1 limitation.

## License

© melthedev. All rights reserved.

This code is not licensed for use, copying, modification, or distribution without permission.

---

A project by **melthedev** — part of the **Moonlit** community.
