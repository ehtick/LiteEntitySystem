---
description: EntityFlags in full - which entities tick where, VisualUpdate scheduling, update order, and the SafeEntityUpdate option.
---

# Update flags and scheduling

`[EntityFlags(...)]` on an entity class decides where its `Update()` and `VisualUpdate()` run and who receives the entity at all — three flags whose combinations cover every scheduling case in the library.

## The flags

| Flag | `Update()` runs | `VisualUpdate()` runs (client) | Synced to |
|---|---|---|---|
| *(none)* | nowhere | never | everyone |
| `Updateable` | server + owning client (prediction) | owning client only | everyone |
| `UpdateOnClient` | server + **every** client | every client | everyone |
| `OnlyForOwner` | *(visibility flag — combine with the above)* | — | owner only |

Two structural facts. `UpdateOnClient` *includes* `Updateable` — it is a superset, not an alternative. And flags OR-merge down the class hierarchy: a subclass inherits every flag of its bases and cannot remove one (which is how a concrete `HumanControllerLogic` ends up both `OnlyForOwner` from `ControllerLogic` and `UpdateOnClient` from its own base).

Client-created local entities (predicted spawns) are always updated on their client regardless of flags.

## Minimal example

The two patterns that cover nearly everything:

**Flags in practice**

```csharp
using LiteEntitySystem;

// Server logic; the owner's client predicts it. Remote clients
// only interpolate SyncVars — no Update, no VisualUpdate.
[EntityFlags(EntityFlags.Updateable)]
public class Crate : EntityLogic
{
    public Crate(EntityParams entityParams) : base(entityParams) { }

    protected override void Update() { /* server + owner prediction */ }
}

// Needs VisualUpdate on every client (a view that moves each frame),
// so it takes UpdateOnClient — and guards its logic accordingly.
[EntityFlags(EntityFlags.UpdateOnClient)]
public class Missile : EntityLogic
{
    public Missile(EntityParams entityParams) : base(entityParams) { }

    protected override void Update()
    {
        if (IsClient && IsRemoteControlled)
            return; // simulate only where authoritative or predicted

        // movement, collision…
    }

    protected override void VisualUpdate()
    {
        // runs on every client, every render frame — move the view here
    }
}
```

## How it works

### Who is "alive"

Internally the manager keeps an alive set — entities whose `Update`/`VisualUpdate` are scheduled. On the server that is every `Updateable` entity. On the client it is: local (predicted-spawn) entities, all `UpdateOnClient` entities, and `Updateable` entities the local player owns. Everything else on the client is passive: it receives state and interpolates, nothing more.

### The UpdateOnClient trade

`VisualUpdate` is only scheduled for alive entities — so a remote entity that must run per-frame view code needs `UpdateOnClient`. The price is that its `Update()` now also runs on clients that do *not* simulate it authoritatively, which is why the `Missile` guards with `IsClient && IsRemoteControlled`. This pairing — flag plus guard — is the current idiom, taken directly from the example project's projectile. For entities whose view can be driven from outside (a proxy object reading `InterpolatedValue`, as in [first-synced-entity.md](../getting-started/first-synced-entity.md)), plain `Updateable` avoids the whole question.

### Update order

Within a tick, alive entities update in creation order (each entity carries an `UpdateOrderNum` assigned at spawn). This order is stable but is not a design surface: don't encode gameplay dependencies in spawn order — use `BeforeControlledUpdate` for controller-before-pawn, and `OnLateConstructed` for construction-time dependencies.

### SafeEntityUpdate

On the server, setting `SafeEntityUpdate = true` wraps every entity's `Update()` in a try/catch: an exception in one entity is logged and skips only that entity, instead of aborting the whole tick. Costs a little performance; useful in production where one broken entity should not stall the world.

## Behavior details

# [Server](#tab/server)

All `Updateable` entities tick every logic tick, in creation order. `VisualUpdate` never runs on the server. `OnlyForOwner` affects only what is sent — the server itself simulates the entity normally.

# [Client](#tab/client)

The alive set defines both `Update` (per tick) and `VisualUpdate` (per render frame). A passive remote entity costs nothing per frame — its fields change only when states arrive. Ownership changes move entities in and out of the alive set automatically.

# [Prediction / rollback](#tab/prediction-rollback)

Rollback re-simulates locally controlled alive entities; flags don't widen that set — an `UpdateOnClient` entity that is remote-controlled is *not* re-simulated, its guarded `Update` simply keeps returning early. Predicted local entities skip re-simulation of ticks before their creation.

Inside `Update` you can always ask which mode you are in: `EntityManager.InRollBackState` / `EntityManager.InNormalState` (or the underlying `EntityManager.UpdateMode`). Gate one-shot side effects — sounds, particles, camera shake — with `InNormalState`, or rollback re-simulation will fire them again on every correction.

***

> [!WARNING]
> **Common mistakes**
>
> * No `[EntityFlags]` at all — the entity syncs but never ticks anywhere; the most common "why is my Update not called".
> * Expecting `VisualUpdate` on a remote entity with plain `Updateable` — it is not in the client's alive set; either take `UpdateOnClient` with the guard, or drive the view externally from `InterpolatedValue`.
> * `UpdateOnClient` without the `IsClient && IsRemoteControlled` guard — clients run simulation logic for entities they do not own and have no inputs for, fighting the incoming server state.
> * Encoding gameplay dependencies in entity creation order — it is stable, but it is bookkeeping, not a contract.

## Related pages

- [singletons.md](singletons.md)
- [../getting-started/first-synced-entity.md](../getting-started/first-synced-entity.md)
