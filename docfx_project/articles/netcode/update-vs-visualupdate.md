---
description: Splitting code between the logic tick and the render frame - what belongs where, and why VisualUpdate is optional.
---

# Update vs VisualUpdate

`Update()` runs on the fixed logic tick and is the only place gameplay state may change; `VisualUpdate()` runs once per render frame on the client and exists purely for presentation — and it is optional, because engine code can read interpolated values just as well.

## The split

| | `Update()` | `VisualUpdate()` |
|---|---|---|
| Runs | every logic tick | every render frame (client) |
| Where | server, and clients that simulate the entity | client only |
| Time source | `DeltaTimeF` (fixed) | `VisualDeltaTimeF` (variable) |
| May change SyncVars | yes | no |
| Re-runs during rollback | yes | never |

Anything the server must agree with goes in `Update`. Anything that only affects what the player sees goes in the render frame — whether that is `VisualUpdate` or your engine's own per-frame callback.

## VisualUpdate is optional

The library never requires it. Reading an interpolated value from engine code works exactly as well:

**PlayerProxy.cs (Unity)**

```csharp
public class PlayerProxy : MonoBehaviour
{
    public MyPlayer Entity;

    // ordinary Unity Update, no entity involvement
    private void Update() => transform.position = Entity.Position.InterpolatedValue;
}
```

Godot's `_Process`, MonoGame's `Draw` or any other per-frame hook does the same job. Views built this way keep engine code in engine classes, which is usually the cleaner split — and it is what the example project does for player transforms.

Reach for `VisualUpdate` when the ordering matters: it runs inside the manager's update, after logic ticks and in entity creation order, so every entity's visual state is computed from the same freshly advanced interpolation, before your engine's own per-frame code runs. Driving a camera rig that must not lag one frame behind its target, or chained visual dependencies, is where that guarantee earns its place. Note that `VisualUpdate` is only called for entities in the client's alive set, so an entity that is not locally simulated needs [`EntityFlags.UpdateOnClient`](../world/update-flags.md) to receive it — one more reason to prefer engine-side view code for simple cases.

## Why logic must stay out of the render frame

Render frames are not part of the simulation. They happen at a variable rate, they are never re-run during rollback, and they do not exist at all on the server. A `SyncVar` written from a render frame is therefore written a variable number of times per tick, is not reproduced when the client re-simulates, and has no counterpart in the server's run of the same tick — three independent ways to diverge from the server.

The one input-related exception is polling: engine input must be sampled per frame, in `VisualUpdate` (or engine code) and written into the pending input struct, precisely because it must *not* be re-sampled during re-simulation. See [input in depth](input-in-depth.md).

> [!WARNING]
> **Common mistakes**
>
> * Changing SyncVars from `VisualUpdate` or engine `Update` — the change is not part of the simulation and diverges from the server.
> * Using `VisualDeltaTimeF` in logic or `DeltaTimeF` in view code — the first breaks determinism, the second makes motion stutter at odd frame rates.
> * Expecting `VisualUpdate` on an entity the client does not simulate — it needs `EntityFlags.UpdateOnClient` to be in the alive set.
> * Sampling input in `Update` — ticks and frames are not 1:1, and rollback would re-sample it; poll per frame, apply per tick.

## Related pages

- [input-in-depth.md](input-in-depth.md)
- [interpolation.md](interpolation.md)
