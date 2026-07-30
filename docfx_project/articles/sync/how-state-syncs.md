---
description: The state synchronization model in brief - baseline plus deltas, RPCs ordered with state, and the consequences for game code.
---

# How state gets to clients

The server periodically snapshots changed entity state and sends it; this page covers only what game code must know about that model — the wire format itself is an implementation detail.

## The model

A joining client first receives a **baseline**: the full state of everything it may see, sent reliably and compressed. From then on it receives **deltas** — only the fields that changed since the state that client last confirmed — sent unreliably at the server's send rate. A lost delta is not retransmitted; the next one carries whatever is still out of date. If a client falls too far behind to be patched incrementally, the server sends a fresh baseline and it resynchronizes silently.

RPCs travel in the same stream, ordered relative to state, and are effectively reliable ordered: each call reaches its recipients exactly once, in order, even though the deltas carrying them are not individually retransmitted. An RPC fired on the tick a field changed arrives with that change applied — never before, never after. Calls fired before a client's current session are not replayed to it.

Entity creation and destruction are part of this stream too, which is why a client's world is always internally consistent: it never sees an entity's data before the entity exists, or a parent link to something that hasn't arrived.

## What this means for game code

**State is sampled, not streamed.** Clients observe values at the send rate, not at every tick. A `SyncVar` that changes and changes back between two sends is never seen changing — if an event matters, send an [RPC](rpc-basics.md); `SyncVar` answers "what is it now", not "what happened".

**Late joins are free.** Any client, joining at any moment, gets a complete world from the baseline. There is no "spawn history" to replay and no code to write for it.

**Bandwidth scales with change, not with world size.** A static entity costs nothing per tick after its creation; a field that changes every tick costs its size every send. This is the lever behind [sync groups](sync-groups.md) and the [flags](sync-flags.md) that restrict who receives what.

**Ordering is guaranteed, timing is not.** Rely on "this RPC lands with that state"; never on a fixed number of ticks or milliseconds between the server's change and the client's reaction.

## Related pages

- [syncvar.md](syncvar.md)
- [rpc-basics.md](rpc-basics.md)
