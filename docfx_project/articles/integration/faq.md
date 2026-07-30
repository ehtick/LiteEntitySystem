---
description: Frequent questions and first-hour mistakes, each with the short answer and a link to the page that covers it.
---

# FAQ and troubleshooting

Answers to the questions that come up most often, grouped by where they bite. Each points at the page with the full story.

## Setup

**The client is rejected at join, or entities arrive as the wrong class.**
Registration differs between builds. Both sides must register the same types under the same enum values; compare `EvaluateEntityClassDataHash()` in your join packet and refuse mismatches. Enum members must never be reordered between releases — append instead. See [registering entity types](../getting-started/registering-entity-types.md).

**`SyncVar<Vector2>` throws at startup.**
Engine types are unknown until registered: `EntityManager.RegisterFieldType<Vector2>(Vector2.Lerp)`, once per process, on both sides, before creating managers. See [SyncVar](../sync/syncvar.md).

**Synchronization silently stops working for one field.**
Something assigned the wrapper instead of the value (`_health = new SyncVar<byte>()`) or replaced a syncable field instance. Install the shipped Roslyn analyzer so the first case cannot compile, and declare syncable fields `readonly`. See [installation](../getting-started/installation.md).

**Nothing is logged when things go wrong.**
`Logger.LoggerImpl` is unassigned — registration errors, missing `base.RegisterRPC` and lag-compensation misses all report through it.

## Entities

**`Update()` is never called.**
The class needs `[EntityFlags(EntityFlags.Updateable)]`. If it should also tick on clients that do not own it, `UpdateOnClient` — with a guard so those clients skip authoritative logic. See [update flags](../world/update-flags.md).

**Values written on the client get overwritten.**
That is prediction working: only the server's values persist. Writes on entities you own are predictions; writes on entities you don't own need `SyncFlags.AlwaysRollback` to be reset properly. See [controlling rollback](../netcode/rollback-per-field.md).

**A field is always default on other clients.**
It is `OnlyForOwner`, or its [sync group](../sync/sync-groups.md) is disabled for that player — in the first case the data never arrives at all.

**An entity reference points at the wrong entity after a while.**
Ids are reused. Store `EntitySharedReference` and resolve with `GetEntityById`, which checks the version. See [entity lifecycle](../world/entity-lifecycle.md).

**`Destroy()` does nothing on the client.**
By design — only the server destroys synchronized entities.

## Movement and visuals

**Remote entities move in steps.**
Views must read `InterpolatedValue`, and the field must carry `SyncFlags.Interpolated`. See [interpolation](../netcode/interpolation.md).

**Rotation spins the long way around.**
Use `FloatAngle` instead of a raw float — it interpolates across the 360° boundary.

**The local player rubber-bands.**
Prediction disagrees with the server: nondeterminism in predicted `Update` (randomness, wall-clock time, engine frame time, live input polling), or gameplay state changed outside the simulation. See [prediction and rollback](../netcode/prediction-and-rollback.md).

**Effects or sounds play several times.**
One-shot code inside predicted `Update` without `EntityManager.InNormalState` — re-simulation replays it on every correction.

## Input and shooting

**Input is dropped or feels unreliable.**
Poll in the render frame (`VisualUpdate` or engine code) and accumulate flags with `|=`; polling inside `Update` loses presses between ticks and breaks re-simulation. See [input in depth](../netcode/input-in-depth.md).

**Movement stops entirely.**
If the pawn relies on `BeforeControlledUpdate`, its `Update()` override must call `base.Update()` first.

**Shots miss what the player clearly aimed at.**
Test fields need `SyncFlags.LagCompensated`, the check must run inside input processing, and physics colliders must be moved to the rewound values in `OnLagCompensationStart`. See [lag compensation](../netcode/lag-compensation-in-depth.md).

## Architecture

**How do I send something from client to server?**
The input struct for continuous intent, `SendRequest`/`SendRequestStruct` for occasional actions. There are no client-to-server RPCs. See [client requests](../sync/client-requests.md).

**How do I keep server-only code out of the client build?**
Register a server subclass and a client subclass of a shared base under the same class id, in separate assemblies. See [registering entity types](../getting-started/registering-entity-types.md).

**How do I handle reconnects without losing the player's character?**
`StopControl()` before `RemovePlayer` so the pawn survives, then create a new controller on reconnect and `StartControl` the old pawn. Keep persistent state on the pawn, never on the controller. See [controllers in depth](../world/controllers-in-depth.md).

**Can I run server and client in one process?**
Yes — the standard listen-server pattern, with the local client connecting over localhost. See [hosting](hosting.md).

**How do I transfer a large file?**
Not through the entity system: use your own channel on the same [transport](transport.md), where you can show progress and throttle.

## Related pages

- [../netcode/diagnostics.md](../netcode/diagnostics.md)
- [limits.md](limits.md)
