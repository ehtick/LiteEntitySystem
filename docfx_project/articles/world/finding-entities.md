---
description: Live entity queries with GetEntities and EntityFilter, spawn/despawn subscriptions, and resolving EntitySharedReference safely.
---

# Finding entities

The manager answers three questions: "all entities of a type" (`GetEntities<T>` — a live filter), "tell me when one appears or dies" (filter subscriptions), and "the entity behind this reference" (`GetEntityById` for `EntitySharedReference`).

## When to use this

* Iterating everything of a kind — all players for a scoreboard, all pickups for a minimap.
* Reacting to spawns and despawns without polling — `SubscribeToConstructed` / `OnDestroyed`.
* Storing entity references inside game state — always as `EntitySharedReference`, resolved on use.

## When not to

* One-per-world services — [`GetSingleton<T>()`](singletons.md) is direct and typed.
* The local player's controller — `GetPlayerController<T>()` on the client, `GetPlayerController(player)` on the server.
* Hand-rolled static registries of entities — a filter already is that list, maintained for you.

## Minimal example

A client-side minimap:

**Minimap.cs (client)**

```csharp
using LiteEntitySystem;

public class Minimap
{
    public Minimap(ClientEntityManager manager)
    {
        EntityFilter<BasePlayer> players = manager.GetEntities<BasePlayer>();
        players.SubscribeToConstructed(AddBlip, callOnExisting: true);
        players.OnDestroyed += RemoveBlip;
    }

    public void Refresh(ClientEntityManager manager)
    {
        foreach (var player in manager.GetEntities<BasePlayer>())
            MoveBlip(player, player.Position);
    }

    private void AddBlip(BasePlayer player) { /* create marker */ }
    private void RemoveBlip(BasePlayer player) { /* remove marker */ }
    private void MoveBlip(BasePlayer player, object pos) { /* update marker */ }
}
```

## How it works

### Filters are live queries

`GetEntities<T>()` returns an `EntityFilter<T>` — not a snapshot. It is created lazily, backfilled with matching entities that already exist, and maintained as entities construct and die. Iteration order is creation order. Matching includes subclasses: `GetEntities<BasePlayer>()` yields `ServerPlayer`/`ClientPlayer` instances from the [side-variant pattern](../getting-started/registering-entity-types.md) as well as any other derived class.

### Spawn and despawn callbacks

`SubscribeToConstructed(callback, callOnExisting)` fires for every entity of the type once it is fully constructed and synced; `callOnExisting: true` replays it for entities already in the filter — essential on the client, where the baseline may have constructed the world before your subscription. `OnDestroyed` fires as entities are removed from the filter; `UnsubscribeToConstructed` detaches.

### Entity references

`EntitySharedReference` is the storable form of "that entity": id plus version, implicit conversion from an entity, `Empty`/`IsValid` for null-checks. Resolve with `GetEntityById<T>(reference)` — it returns null for an empty reference, a type mismatch, or a stale version after id reuse — or with `TryGetEntityById<T>(reference, out var entity)`:

```csharp
public SyncVar<EntitySharedReference> LastAttacker;

if (EntityManager.TryGetEntityById(LastAttacker, out BasePlayer attacker))
    ShowKillerName(attacker);
```

Resolve-on-use is the pattern: the reference lives in state, the object reference lives on the stack.

## Behavior details

# [Server](#tab/server)

Filters see the whole world — every synced entity plus server-only AI controllers. The library's own loops (input application over `GetEntities<HumanControllerLogic>`, for instance) run on the same filters your code uses.

# [Client](#tab/client)

Filters contain only what this client receives: no other players' controllers, no entities hidden by `EntityFlags.OnlyForOwner`, no AI controllers. The same `GetEntities<BasePlayer>()` loop is simply shorter here — code shared between sides needs no special casing.

# [Prediction / rollback](#tab/prediction-rollback)

References resolve identically during rollback, and local predicted entities live in the filters like everything else. Filter membership itself is not rolled back — an entity is in or out based on its actual construction state, not the re-simulated tick.

***

> [!WARNING]
> **Common mistakes**
>
> * Keeping plain C# references to entities in long-lived state — after destruction and id reuse they point at the wrong object; store `EntitySharedReference`, resolve on use.
> * Destroying entities while iterating their filter — copy to a temporary array first (the library does exactly this in its own bulk operations).
> * Subscribing on the client with `callOnExisting: false` — everything constructed from the baseline before your subscription is silently missed.
> * Expecting client filters to match the server's — the client sees only its slice of the world; counts and iteration results legitimately differ per side.

## Related pages

- [unity-views.md](unity-views.md)
- [singletons.md](singletons.md)
