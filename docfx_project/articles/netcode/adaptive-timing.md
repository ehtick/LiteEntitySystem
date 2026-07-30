---
description: The elastic client clock and the two buffers it feeds - what jitter does, which values to watch, and when tuning is warranted.
---

# Adaptive timing and buffers

Packets never arrive evenly. To turn an uneven stream into smooth playback, the client keeps two small buffers and continuously stretches or compresses its own clock to keep them at a healthy depth — automatically, with a few knobs for extreme cases.

## The two buffers

**The state buffer** holds server states the client has received but not yet played. Interpolation of remote entities blends between the two states at its head, so this buffer is what makes remote motion smooth; if it drains to empty, there is nothing to interpolate towards and remote entities stall.

**The server-side input buffer** holds the client's inputs that have arrived but whose tick the server has not simulated yet. If it runs dry, the server has no input for that tick and repeats the last one — the player's controls appear to stick.

Both need to be non-empty but shallow: depth is latency. Every extra buffered state is extra delay before the player sees what happened; every extra buffered input is extra delay before their action takes effect.

## The elastic clock

The client cannot change the server's rate, but it can change its own — so it speeds its clock up by a fraction to drain a buffer that is too deep, and slows it down to let a shallow one refill. The adjustment is small, continuous and unnoticeable; the effect is that both buffers hover around the target depth instead of oscillating between starving and bloating.

The target itself follows measured jitter: the noisier the connection, the deeper the buffers must be to survive it. `PreferredBufferTimeLowest` and `PreferredBufferTimeHighest` set the bounds added on top of the measured jitter — the defaults (25 ms and 50 ms) suit ordinary internet play.

Nothing here is manual: no game code reads or drives the clock. It matters only when reading diagnostics, or when the defaults do not fit the network you ship on.

## What to watch

| Value | Meaning | Healthy |
|---|---|---|
| `NetworkJitter` / `AverageJitter` | Measured arrival irregularity | Stable, low; spikes explain buffer growth |
| `LerpBufferCount` / `LerpBufferTimeLength` | Depth of the state buffer, in states and in seconds | Small and steady, never persistently zero |
| `ServerInputBuffer` | Inputs queued on the server for this client | A couple of ticks, never zero for long |
| `StoredCommands` | Unconfirmed inputs held for rollback | Grows with round-trip time; sustained growth means the server is not confirming |
| `StateSize` | Bytes in the current state | Watch when adding entities or fields |

An overlay showing these is the fastest way to explain "it feels laggy" — see [debugging and diagnostics](diagnostics.md).

## When to tune

Leave the defaults unless measurements say otherwise.

* **High-jitter networks** (mobile, congested Wi-Fi): raise `PreferredBufferTimeLowest`/`Highest` to trade a little latency for fewer stalls.
* **LAN or tightly controlled deployments**: lower them for a slightly more responsive game, accepting that any real jitter will now be visible.
* **Send rate first.** Buffer depth is measured in states; halving [`ServerSendRate`](tick-model.md) doubles the time each buffered state represents. If remote motion looks choppy, check the send rate before touching buffer bounds.

On the server side, `PlayerResyncTimeout` (4 seconds by default) decides how long a client may fail to acknowledge states before it is resynchronized with a fresh baseline instead of more deltas — relevant on lossy connections, not something to tune casually.

> [!WARNING]
> **Common mistakes**
>
> * Treating a persistently empty state buffer as a tuning problem — an empty buffer usually means states are not arriving (loss, bandwidth), not that the bounds are wrong.
> * Setting the preferred buffer times to zero for "minimum latency" — the buffers then starve on the first jitter spike, and remote entities stall visibly.
> * Comparing the client's `Tick` to `ServerTick` to estimate delay — they are independent counters; use the diagnostics above.
> * Chasing input latency by tuning buffers rather than raising the [tick rate](tick-model.md) — the tick rate is the larger term.

## Related pages

- [diagnostics.md](diagnostics.md)
- [tick-model.md](tick-model.md)
