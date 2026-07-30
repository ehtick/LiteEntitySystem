---
description: The five ideas behind every LiteEntitySystem game - entities, fixed ticks, server authority, inputs up and state down, prediction with interpolation.
---

# Core model in five minutes

Every feature in LiteEntitySystem is a consequence of five ideas; this page walks through them once so the rest of the documentation can build on them without re-explaining.

## The world is a set of entity classes

A game world is a collection of entity objects: players, projectiles, items, score counters. Entity classes are plain C# — no engine base classes — and both server and client register the same set of classes under the same ids, then construct instances of them. The server decides *which* entities exist; each client holds a synchronized copy of the entities it is allowed to see.

Visuals are separate. An entity may create engine objects (a Unity GameObject, a sprite, a sound) as its *view*, typically when it is constructed, but the view is game code — the library synchronizes only entity state.

## Time advances in fixed ticks

All game logic runs at a fixed tick rate (`Tickrate`), chosen when the server starts. Once per tick the manager calls `Update()` on entities that update — this is the logic tick, and it is the only place where gameplay state should change. You pump the manager's own `Update()` from your engine loop every render frame; it accumulates real time and runs zero or more logic ticks internally.

Rendering is decoupled from ticks. Entities also get `VisualUpdate()` once per render frame on the client — that is where views read interpolated values. The current tick number is `Tick` (a `ushort`, it wraps around).

## The server owns all state

The server's simulation is the single source of truth. Changed `SyncVar` fields are collected and sent to clients as delta-compressed state updates at the configured send rate; a joining client first receives the whole world as one compressed baseline, then only the changes. RPCs travel in the same stream, ordered relative to state. The consequences of that model — sampling, late joins, bandwidth — are on [how state gets to clients](../sync/how-state-syncs.md).

A client cannot create or destroy synchronized entities and cannot make a lasting change to a `SyncVar`. Anything a client writes is either a prediction (corrected by the next server state) or a purely local change.

## Clients send inputs, not state

The only thing a client continuously uploads is its input: one small unmanaged struct per tick, delta-compressed, resent until the server confirms it. The server applies each input on the matching tick of its own simulation and moves the player's entities itself.

This is why there are no client-to-server RPCs. For occasional non-input communication — buy an item, change loadout — there is a reliable request channel with a server response callback.

## The client predicts and interpolates

For entities owned by the local player, the client does not wait for the server: it applies its own input immediately, simulating those entities ahead of server time (client-side prediction). When a server state arrives, predicted fields are reset to the authoritative values and the entity is re-simulated from stored inputs — a rollback. If the prediction was right, the correction is invisible.

Entities owned by others are shown slightly in the past instead: the client keeps a small buffer of server states and interpolates between the last two. So one client screen mixes two timelines — your own entities ahead of the server, everyone else behind it — and lag compensation on the server exists to reconcile the difference when you shoot at what you see.

## Data flow at a glance

```
            inputs: 1 struct per tick, delta-compressed, unreliable
   ┌────────────────────────────────────────────────────────────────►
CLIENT                                                           SERVER
 predicts own entities                                    fixed-tick simulation,
 interpolates the rest                                    single source of truth
   ◄────────────────────────────────────────────────────────────────┐
            on join: full world baseline (LZ4, reliable)
            then:    state diffs + RPCs (at send rate, unreliable)
```

## One cycle, side by side

# [Server tick](#tab/server-tick)
Read buffered client inputs for this tick → run `Update()` on entities → record history for lag compensation → every send-rate ticks, serialize changed fields per player and send diffs and pending RPCs.
# [Client frame](#tab/client-frame)
Pump manager `Update()` → run due logic ticks (capture input, predict own entities, send input packet) → advance interpolation between the two buffered server states → call `VisualUpdate()` on entities so views read interpolated values.
# [Client, on state arrival](#tab/client-on-state-arrival)
Execute RPCs ordered with the state → reset predicted fields to server values → re-simulate own entities from stored inputs (rollback) → spawn/destroy entities the server reported → the state becomes the new interpolation target. During the re-simulation `EntityManager.InRollBackState` is true — entity code uses it (or `InNormalState`) to skip one-shot effects that would otherwise replay on every correction.
***

> [!WARNING]
> **Common mistakes**
>
> * Changing gameplay state in `VisualUpdate()` — it runs per render frame, is never predicted or rolled back, and diverges from the server. Logic belongs in `Update()`.
> * Expecting a server-side `SyncVar` write to be visible on clients in the same instant — it arrives with the next state update, delayed by send rate and network latency.
> * Treating the client's `Tick` as server time — it is an independent counter that starts from zero at connection; the server timeline on the client is `ServerTick`.

## Related pages

- [installation.md](installation.md)

- [first-synced-entity.md](first-synced-entity.md)
