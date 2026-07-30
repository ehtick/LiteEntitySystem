---
description: Lag compensation mechanics - how the rewind target is derived, history sizing, physics integration, client-side rewind during rollback, and misses.
---

# Lag compensation in depth

The [getting-started page](../getting-started/lag-compensation.md) covers the API; this one covers what the rewind actually does — which moment it reconstructs, how far back it can reach, and why the same call behaves differently on the client.

## Reconstructing the shooter's view

The server does not rewind by "the shooter's ping". Every input a client sends carries the two server states it was interpolating between at that moment, plus the blend factor. When the server processes that input, it has an exact description of the frame the player was looking at, and rewinds lag-compensated fields to that reconstructed moment — the same blended position the shooter saw, not a snapshot boundary.

This is why compensation is tied to input processing: the rewind target comes from the input's own metadata. A hit check run outside input processing has nothing to reconstruct.

## History and its bounds

Each lag-compensated field is recorded every tick into a ring buffer of `MaxHistorySize` ticks (16, 32, 64 or 128; the default is 32). Two limits follow:

* **Reach.** 32 ticks is roughly half a second at 60 ticks per second, a full second at 30. A shot whose reconstructed moment is older than the buffer cannot be compensated.
* **Cost.** Memory is `history size × field size × number of lag-compensated entities`, written every tick. Marking more fields than the hit checks need is a per-tick cost for nothing.

When the target moment has fallen out of the window, the library logs a compensation miss and the check runs against present-time values — the shot is evaluated, just without the rewind.

Choosing the size is a design decision, not a technical one: longer history honors more shots from laggy players, at the price of more "I was already behind cover" deaths for their victims. Shorter history does the reverse.

## Integrating with physics

Rewinding restores field values, not engine state. If the hit check goes through a physics engine, each rewound entity gets `OnLagCompensationStart()` and `OnLagCompensationEnd()`:

**Fighter.cs**

```csharp
protected override void OnLagCompensationStart()
{
    // push rewound values into the engine so queries see the old pose
    _body.position = new Vector2(X.Value, Y.Value);
}

protected override void OnLagCompensationEnd()
{
    // values are restored to the present; sync the engine back
    _body.position = new Vector2(X.Value, Y.Value);
}
```

Engines that batch transform updates may also need an explicit sync (`Physics2D.SyncTransforms`, or the equivalent) before querying. Keep the enable/query/disable window tight: while it is open, the world's rewound entities are in the past, and anything else running in that window sees a false present.

Note that interpolated fields read `.Value` — not the smoothed copy — while compensation is active, so a physics push using `.Value` is correct in both hooks.

## On the client

`EnableLagCompensationForOwner()` is a no-op during normal client simulation: the client already sees its own screen, and there is nothing to reconstruct.

During rollback re-simulation it does work. The client replays its stored inputs, and each carries the same state/blend metadata the server will use — so the client's hit checks rewind to exactly the moment the server will reconstruct. That is what makes predicted hits agree with the server's verdict, and why shared shooting code should call the enable/disable pair unconditionally rather than guarding it with `IsServer`.

## When compensation is not the answer

* **Slow projectiles.** A rocket that travels for seconds is simulated per tick; each tick's collision check can be compensated, but the projectile's own position is state, not a rewind problem. See [predicted spawning](predicted-spawning.md).
* **Area effects over time.** A lingering damage field has no single "moment the player aimed"; evaluate it in the present.
* **The shooter's own entity.** Never rewound — the owner is excluded by design.

> [!WARNING]
> **Common mistakes**
>
> * Hit checks outside input processing (a timer, a delayed callback) — there is no input metadata to reconstruct a moment from, so nothing meaningful is rewound.
> * Marking fields `LagCompensated` that no check reads — history is written for them every tick regardless.
> * Reading `InterpolatedValue` inside the compensation window — compensation moves `.Value`; the smoothed copy is for rendering.
> * Guarding the enable/disable pair with `IsServer` — the client needs it during rollback for predicted hits to match.

## Related pages

- [predicted-spawning.md](predicted-spawning.md)
- [../getting-started/lag-compensation.md](../getting-started/lag-compensation.md)
