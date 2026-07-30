---
description: The built-in ceilings - entity and player counts, buffers and payload sizes - and what happens when a game approaches them.
---

# Limits and constants

The library has fixed ceilings that come from the wire format: ids are 16 bits, player ids 8, payload sizes 16. This page lists them so you can size your game against them.

> [!CAUTION]
> Treat these numbers as boundaries to stay away from, not as budgets to fill. Behavior at the edge is not uniformly graceful — some limits are reported clearly, others degrade or fail in ways that are hard to diagnose. A design that runs at 90% of any of them is a design that will break in production.

## The ceilings

| Limit | Value | What it bounds |
|---|---|---|
| `MaxSyncedEntityCount` | 64000 | Synchronized entities alive at once, server-wide |
| `MaxEntityCount` | 65534 | Total id space; the range above the synced count is for client-local predicted entities |
| `MaxLocalEntityCount` | ~1534 | Client-local predicted entities alive at once |
| `MaxPlayers` | 254 | Simultaneous players (id 0 is the server, ids 1-254 are players) |
| `MaxStoredInputs` | 30 | Inputs the server buffers per player |
| `InputBufferSize` | 64 | Unconfirmed inputs the client keeps for rollback |
| `MaxHistorySize` | 16 / 32 / 64 / 128 ticks | Lag-compensation history depth (constructor parameter) |
| RPC payload | 64 KB | Bytes in a single RPC's payload |
| `StringSizeLimit` | 1024 | Bytes in a string inside a client request |

`MaxHistorySize` is the only one you choose; the rest are properties of the format.

## What running close to a limit looks like

**Entities.** The practical constraint is bandwidth long before 64000: every synchronized entity contributes to state size, and clients receive a full baseline of everything they can see when they join. If entity counts are the concern, [sync groups](../sync/sync-groups.md) reduce what each player receives; culling never reduces the server-side count.

**Players.** 254 is the id space, not a performance figure. State is composed per player, so the cost of a player is the size of their view of the world — a shooter and a hundred-player battle royale have very different real ceilings on the same hardware.

**Inputs.** The server's per-player buffer is deliberately shallow: it exists to absorb jitter, not to store history. A client whose clock has drifted far enough to overrun it is a client that will be corrected by the [adaptive timing](../netcode/adaptive-timing.md) machinery — that is the mechanism working, not a limit to raise.

**Payloads.** The 64 KB RPC ceiling is far above what a single call should carry. Anything approaching it belongs in a [syncable field](../sync/syncablefield-basics.md) that synchronizes incrementally, or should be split into several calls; a payload larger than the connection's packet size is split across packets and delays everything queued behind it.

For genuinely large transfers — a level file, a downloadable map, a replay blob — do not use the entity system at all. Send them on your own channel through the [transport](transport.md), sharing the connection via the header byte: LiteNetLib (or whatever transport you use) can stream them with its own chunking, and you keep what the entity system cannot give you — a progress figure to show a loading bar, cancellation, and the freedom to throttle the transfer so it does not compete with gameplay traffic. The entity stream is for game state; bulk content belongs beside it.

## Designing away from the edges

* Destroy what is no longer needed — entities count against the ceiling until they are actually removed, which happens after all clients have acknowledged the destruction.
* Prefer a few rich entities over many trivial ones: each has fixed per-entity overhead in every state that mentions it.
* Keep predicted local entities short-lived; the local id space is much smaller than the synced one.
* Measure with the [diagnostics](../netcode/diagnostics.md) values (`EntitiesCount`, `StateSize`) rather than reasoning from the table above — the number that actually hurts is bytes per second per player, and it is reachable well below every limit here.

## Related pages

- [faq.md](faq.md)
- [../netcode/diagnostics.md](../netcode/diagnostics.md)
