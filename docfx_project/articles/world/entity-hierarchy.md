---
description: The entity base classes - what each level of the hierarchy adds, and a decision table for choosing what to subclass.
---

# Entity class hierarchy

Every networked object descends from `InternalEntity`, but game code always subclasses one of five specialized bases — this page shows what each one adds and how to pick.

```
InternalEntity                     id, ownership, lifecycle, RPC
├─ EntityLogic                     world objects: hierarchy, sync groups, predicted spawn
│  ├─ PawnLogic                    possessable by a controller
│  └─ PredictableEntityLogic       spawnable inside client prediction
├─ ControllerLogic                 possesses pawns, synced only to its owner
│  ├─ AiControllerLogic<T>         server-only bot brain
│  └─ HumanControllerLogic<TInput> player input pipeline
└─ SingletonEntityLogic            exactly one instance, server-owned
```

## Choosing a base

| You are building | Subclass | Why |
|---|---|---|
| Pickup, door, zone, item — a world object | `EntityLogic` | Full entity: SyncVars, hierarchy, RPCs. |
| A player's body (character, vehicle) | `PawnLogic` | Adds possession; ownership follows the controller. |
| A player's input handling | `HumanControllerLogic<TInput, TPawn>` | Input pipeline + prediction support; exists only on the owner's client and server. |
| A bot's decision making | `AiControllerLogic<TPawn>` | Same possession model, server-only, never synced. |
| A projectile the client spawns predictively | `PredictableEntityLogic` | Spawned via `AddPredictedEntity`, matched to the server copy later. |
| Match state, physics service — one per world | `SingletonEntityLogic` | Single instance, `GetSingleton<T>()` lookup, always server-owned. |
| A non-networked local service | none — `ILocalSingleton` | Lives beside the world, not in it. |

## What each level adds

### InternalEntity — the root

Identity (`Id`, `ClassId`, `Version`), ownership (`OwnerId`, `IsLocalControlled`, `IsRemoteControlled`, `IsServerControlled`), side checks (`IsServer`/`IsClient`, `ClientManager`/`ServerManager` accessors), the lifecycle overrides (`OnConstructed`, `OnLateConstructed`, `OnDestroy`, `Update`, `VisualUpdate`, rollback hooks) and the RPC machinery (`RegisterRPC`, `ExecuteRPC`). It lives in the `LiteEntitySystem.Internal` namespace but is fully public API — you never subclass it directly, because no spawn method accepts a direct descendant.

### EntityLogic — world objects

Everything meant to exist *in* the world: the parent/child tree (`SetParent`, `Childs`, destruction cascades), per-player sync groups (`IsSyncGroupEnabled`, `OnSyncGroupsChanged`), predicted spawning of child entities (`AddPredictedEntity`), scoped lag compensation (`EnableLagCompensationForOwner`) and `SharedReference` for storing references in SyncVars.

### PawnLogic — possessable bodies

A thin but important layer: the `Controller` property and the `Update()` that lets the possessing controller act first ([adding-a-player.md](../getting-started/adding-a-player.md)). Possession rewires ownership — the pawn becomes owned by the controlling player, which switches on prediction for that client.

### ControllerLogic — the brains

Controllers hold `StartControl`/`StopControl`/`GetControlledEntity` and are marked "only for owner": other players never receive them, so whatever a controller stores stays between the server and the owning client. `HumanControllerLogic<TInput>` adds the input struct pipeline and the reliable request channel; `AiControllerLogic` is its server-only sibling for bots — same possession model, zero network cost. The `<TInput, TPawn>` / `<TPawn>` variants add a typed `ControlledEntity` shortcut.

### PredictableEntityLogic — predicted spawns

For entities that must appear on the owning client the instant they are created — projectiles above all. Created only through `AddPredictedEntity`, matched to the authoritative server copy when it arrives (`IsRecreated`, `OnEntityRecreated`). Covered in depth in the prediction section.

### SingletonEntityLogic — one per world

For synchronized services where a second instance is a bug: match rules, the physics manager. Always server-owned, found via `GetSingleton<T>()`/`TryGetSingleton`, spawned with `AddSingleton<T>()`. Not to be confused with `ILocalSingleton` — a plain interface for *non-networked* per-manager services.

## Server and client variants

Any of these bases can additionally be split into side variants: a shared class holding all synchronized members, with a server subclass and a client subclass registered under the same class id. The pattern and its rules live in [registering-entity-types.md](../getting-started/registering-entity-types.md); it applies to the whole hierarchy, controllers included.

> [!WARNING]
> **Common mistakes**
>
> * Subclassing `InternalEntity` directly — it compiles, but no spawn method (`AddEntity`, `AddController`, `AddSingleton`, `AddPredictedEntity`) accepts it; use one of the five bases.
> * Storing owner-private data (aim state, cooldown internals, loadout secrets) in pawn SyncVars — the pawn syncs to everyone; data that only the owner should see belongs in the controller.
> * Modeling a world-wide service as a plain `EntityLogic` — nothing stops a second instance; `SingletonEntityLogic` encodes the invariant and gives `GetSingleton` lookup.

## Related pages

- [entity-lifecycle.md](entity-lifecycle.md)

- [registering-entity-types.md](../getting-started/registering-entity-types.md)
