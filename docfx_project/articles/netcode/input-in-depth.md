---
description: The input pipeline in detail - designing the struct, pending vs current input, buffering on both sides, and redundant delta-compressed sending.
---

# Input in depth

Input is the client's only continuous upward channel: one unmanaged struct per tick, filled during render frames, applied on the logic tick, kept in history for rollback, and sent redundantly so packet loss costs nothing.

## Designing the struct

The struct travels every tick, for every player, and is stored in history on both sides — so it should be small and flat.

**PlayerInput.cs**

```csharp
using System;

[Flags]
public enum MoveKeys : byte
{
    Up = 1, Down = 1 << 1, Left = 1 << 2, Right = 1 << 3,
    Fire = 1 << 4, Jump = 1 << 5, Use = 1 << 6
}

public struct PlayerInput
{
    public MoveKeys Keys;     // pack booleans into flags, not separate bools
    public float Rotation;    // analog values as-is
}
```

Guidelines that follow from how it is transmitted:

* **Flags over booleans.** Eight buttons in one byte, and unchanged bytes cost nothing after delta compression.
* **Intent, not results.** Send "forward is held", not a computed velocity — the server must derive the outcome itself, or it is trusting the client.
* **Quantize what tolerates it.** A rotation is often fine as a `short` in a fixed range; a full `float` is 4 bytes every tick.
* **No references.** The type must be `unmanaged`; entity references travel as `EntitySharedReference`.

If a clean input is not all zeroes for your game — a default weapon slot, a neutral stance — override `GetDefaultInput()` on the controller.

## Pending and current

Two values live in the controller. `ModifyPendingInput()` returns a `ref` to the *pending* struct, which accumulates what the player did since the last tick; at the next logic tick the library promotes it to `CurrentInput`, stores it in the history buffer, and resets pending to the default.

```csharp
protected override void VisualUpdate()
{
    ref PlayerInput input = ref ModifyPendingInput();

    // OR-ing flags across frames: a keypress between ticks is not lost
    if (Engine.IsKeyDown(Key.W))
        input.Keys |= MoveKeys.Up;

    // last-writer-wins for analog values
    input.Rotation = Engine.GetAimAngle();
}
```

Accumulating with `|=` matters: several render frames usually happen per tick, and a button tapped in any of them should survive to the tick. Analog values are naturally last-writer-wins.

Read `CurrentInput` in `BeforeControlledUpdate` (or from the pawn through the controller) — never the pending value, which is the *next* tick's data.

## Buffering and sending

Each tick the client stores its input and sends a packet containing not just that tick's input but a window of recent unconfirmed ones, delta-compressed against each other. A lost packet is therefore covered by the next one, with no retransmission and no added latency — the cost is a few bytes per tick.

On the server, arriving inputs go into a per-player buffer keyed by tick, and each tick consumes the entry matching the tick being simulated. Two consequences:

* **A missing input repeats the last one.** If nothing arrived for the tick, the previously applied input stays in effect — a held key is a better guess than a sudden stop.
* **The buffer is deliberately non-empty.** The client's adaptive clock keeps a small reserve of inputs queued on the server, so jitter does not starve it; `ServerInputBuffer` on the client reports the current depth. See [adaptive timing](adaptive-timing.md).

Stored client-side inputs are dropped once the server confirms the tick that consumed them; the remaining ones are exactly what rollback replays.

## Rollback replay

During re-simulation the library reloads `CurrentInput` from history for each replayed tick, so `BeforeControlledUpdate` and the pawn's `Update` see precisely what they saw the first time. This is the reason polling belongs in the render frame: a live poll inside a re-simulated tick would feed today's input into yesterday's tick and desynchronize the prediction.

> [!WARNING]
> **Common mistakes**
>
> * Polling input in `Update` or `BeforeControlledUpdate` — keypresses between ticks are lost, and rollback re-runs those methods with historical input.
> * Assigning flags instead of OR-ing them in the pending struct — a frame with no keys held wipes what earlier frames in the same tick recorded.
> * Sending computed results (velocity, a hit target) instead of intent — the server has no way to validate them, which is exactly what server authority is for.
> * A large input struct — it ships every tick with redundancy; every field is paid for continuously.

## Related pages

- [prediction-and-rollback.md](prediction-and-rollback.md)
- [../sync/client-requests.md](../sync/client-requests.md)
