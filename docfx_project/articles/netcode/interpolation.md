---
description: Smoothing rendered values - what InterpolatedValue blends between for owned and remote entities, and registering lerp functions for custom types.
---

# Interpolation

State arrives in discrete snapshots, but rendering happens every frame — `SyncFlags.Interpolated` keeps a second, smoothed copy of a field so views can move continuously between the values the simulation produced.

## Using it

Mark the field, read `InterpolatedValue` from view code:

**Drone.cs**

```csharp
[SyncVarFlags(SyncFlags.Interpolated)]
public SyncVar<float> Altitude;
```

```csharp
// per-frame view code (engine Update or VisualUpdate)
mesh.position = new Vector3(0f, drone.Altitude.InterpolatedValue, 0f);
```

Logic keeps reading `.Value` — the authoritative number for this tick. The two are different by design: `.Value` is where the simulation is, `InterpolatedValue` is what the player should see right now.

## What it blends between

The answer differs by ownership, because the two cases have different sources of truth:

| Entity | Blends between | Effect |
|---|---|---|
| Locally controlled (predicted) | the value before the last logic tick and the current one | smooths the client's own simulation across render frames |
| Remote | the two most recent server states | renders slightly behind the server, hiding jitter and send-rate gaps |

So one screen shows two timelines at once: your own entities interpolated across their predicted ticks, everyone else's interpolated between snapshots that are a fraction of a second old. That difference is exactly what [lag compensation](lag-compensation-in-depth.md) reconciles when you shoot.

The blend factors are exposed for special cases: `LerpFactor` reports how far the current frame sits between two logic ticks, and `VisualDeltaTimeF` gives the render-frame delta.

## Custom types

Interpolation needs to know how to blend a type. Primitives are built in, and so is `FloatAngle`, which wraps correctly across 360° — use it for rotations instead of a raw float, which would spin the long way around from 359 to 1.

Engine vectors are registered by your code once per process, on both sides, before any manager exists:

```csharp
EntityManager.RegisterFieldType<Vector2>(Vector2.Lerp);
EntityManager.RegisterFieldType<Vector3>(Vector3.Lerp);
```

Any `unmanaged` type works with a matching `(prev, current, t) => value` function. Registering without one is valid too — the type then synchronizes normally but cannot be marked `Interpolated`.

## Which fields deserve it

Interpolation costs a second copy of the value on the client and a lerp per read, and it only makes sense for quantities that move continuously: position, rotation, a turret angle, a smoothly draining bar. Discrete values — health, ammo, a state enum, a boolean — should snap; a midpoint between 100 and 75 health is a number the game never had.

For remote entities there is a subtler consequence: interpolated fields show a slightly delayed truth. Never use them for logic decisions; logic reads `.Value` on the server, and on the client only for entities it predicts.

> [!WARNING]
> **Common mistakes**
>
> * Views reading `.Value` — motion steps at the send rate instead of flowing; that is the single most common "why is it choppy".
> * Marking discrete values `Interpolated` — the view displays values that never existed.
> * A raw float for angles — interpolating 359 → 1 sweeps backwards through the whole circle; use `FloatAngle`.
> * Forgetting `RegisterFieldType` on one side — the sides disagree about the field's layout and the connection fails at startup rather than subtly.

## Related pages

- [lag-compensation-in-depth.md](lag-compensation-in-depth.md)
- [../sync/sync-flags.md](../sync/sync-flags.md)
