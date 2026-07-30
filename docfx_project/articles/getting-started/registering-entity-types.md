---
description: How to declare the class-id enum, register entity constructors in EntityTypesMap, and reject mismatched builds with the hash handshake.
---

# Registering entity types

`EntityTypesMap<T>` maps your entity classes to numeric class ids taken from an enum; both server and client build this map at startup, and it is what allows the library to construct the right class when an entity syncs in.

## When to use this

* Once at startup, on both sides, before creating a `ServerEntityManager` or `ClientEntityManager` — the manager takes the finished map as its first constructor argument.
* For every class that exists in the synchronized world: entities, pawns, controllers, singletons.
* AI controllers (`AiControllerLogic` subclasses) may be registered on the server only — they are never synchronized to clients.

## When not to

* Pure view classes — effects, UI, camera scripts — are not entities and are never registered; they live entirely in engine code.
* Don't branch registration by game mode or scene: class ids must mean the same thing everywhere. Branching by *side* is allowed only in two supported forms — server-only AI controllers, and server/client subclass variants of a shared base (see below).

## Minimal example

Shared code, executed identically by server and client:

**GameTypes.cs**

```csharp
using LiteEntitySystem;

public enum GameEntities : ushort
{
    Player,
    PlayerController,
    Projectile
}

public static class GameTypes
{
    public static EntityTypesMap<GameEntities> Build() =>
        new EntityTypesMap<GameEntities>()
            .Register(GameEntities.Player, e => new BasePlayer(e))
            .Register(GameEntities.PlayerController, e => new BasePlayerController(e))
            .Register(GameEntities.Projectile, e => new SimpleProjectile(e));
}
```

## How it works

### The class-id enum

The enum is the wire contract: each member's numeric value becomes the entity's class id in every packet. Names are irrelevant to the network — values are not. Renaming a member is safe; reordering or inserting members shifts the values of everything after them and breaks compatibility with older builds.

### Register and the constructor delegate

`Register` ties an enum member to an `EntityConstructor<TEntity>` — a delegate that receives `EntityParams` and returns a new instance, typically just `e => new BasePlayer(e)`. The library calls it on the server when you spawn the entity and on the client when the entity first arrives in a state update. `Register` returns the map, so calls chain.

### One map, two sides

Pass the finished map to the manager constructor. Registration must be identical on both sides: when the server sends an entity with class id 2, the client must have class id 2 registered as the same kind of entity with the same synchronized fields. The single exception is `AiControllerLogic` subclasses — they never sync, so the client may skip them (in the example project, the client registers everything except `BotController`).

### Server and client variants of a type

Under the same enum id, each side may construct its own subclass of a shared entity base:

**Registration per side**

```csharp
// shared assembly: class BasePlayer : PawnLogic { /* all SyncVars, RPCs */ }
// server assembly: class ServerPlayer : BasePlayer { /* server-only logic */ }
// client assembly: class ClientPlayer : BasePlayer { /* views, effects */ }

// on the server:
map.Register(GameEntities.Player, e => new ServerPlayer(e));
// on the client:
map.Register(GameEntities.Player, e => new ClientPlayer(e));
```

The synchronized layout — every `SyncVar`, `SyncableField` and RPC declaration — must stay in the shared base; the side subclasses add only logic. This is the supported way to keep server-only and client-only code in separate assemblies (separate `.asmdef` in Unity), so a client build ships without server logic and vice versa. Queries work through the base class: `GetEntities<BasePlayer>()` returns the `ServerPlayer` or `ClientPlayer` instances, since filters match entities by their base types as well.

### The hash handshake

`EvaluateEntityClassDataHash()` computes a hash over the synchronized layout of every registered type — its `SyncVar` fields and RPC declarations, in class-id order, with server-only AI controllers excluded. Send it in your join packet and compare on the server; on mismatch, disconnect the client instead of letting two incompatible builds desync mid-game. The example does exactly this with its `JoinPacket.GameHash`.

## Behavior details

# [Server](#tab/server)
The map is fixed at manager construction. Spawning a type that was not registered throws an exception immediately (`Unregistered entity type`). Class ids also size the internal per-type serializers, so the map must be complete before the manager exists.
# [Client](#tab/client)
Every incoming entity carries its class id; the client looks up the registered constructor and builds the instance. An unknown or unregistered class id is an error logged through `ILogger` — with identical maps and the hash handshake in place, it should never happen in production.
***

> [!WARNING]
> **Common mistakes**
>
> * Inserting or reordering enum members between releases — every id after the change shifts, and old builds construct wrong classes. Append new members at the end instead.
> * Skipping the hash handshake — incompatible builds then fail with confusing per-entity errors and desync instead of one clean rejection at join.
> * Registering different sets on client and server (other than AI controllers) — the missing side hits `Unregistered entity class` as soon as such an entity syncs.
> * Declaring a `SyncVar` or RPC in a server-only or client-only subclass variant — the synchronized layouts diverge and the hash handshake rejects the connection. Synchronized members belong in the shared base.

## Related pages

- [starting-a-server.md](starting-a-server.md)

- [starting-a-client.md](starting-a-client.md)
