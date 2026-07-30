---
description: Possession API, the BeforeControlledUpdate ordering guarantee, controller lookups, and the disconnect/reconnect pattern that keeps the pawn alive.
---

# Controllers in depth

A controller is the possession point between a player and the world: it can grab and release pawns at runtime, it is private to its owner, and — done right — it is disposable, which is exactly what a clean reconnect flow relies on.

## When to use this

* Runtime possession changes: respawning into a new pawn, entering a vehicle, switching to spectator.
* Disconnect/reconnect handling where the player's body must survive the connection.

## When not to

* Bot logic — that is [ai-controllers.md](ai-controllers.md), same possession model without the network.
* Anything that must outlive the connection (score, inventory, match stats) — that state belongs on the pawn or a game entity, not on the controller (see the reconnect pattern below).

## Minimal example

Server-side session handling that survives a dropped connection:

**PlayerSessions.cs (server)**

```csharp
using LiteEntitySystem;
using LiteEntitySystem.Transport;

public class PlayerSessions
{
    private readonly ServerEntityManager _manager;

    public PlayerSessions(ServerEntityManager manager) => _manager = manager;

    public void OnDisconnected(NetPlayer player)
    {
        // detach first — otherwise RemovePlayer destroys the pawn together
        // with the controller via DestroyWithControlledEntity
        _manager.GetPlayerController(player)?.StopControl();
        _manager.RemovePlayer(player);
        // the pawn stays in the world, reverted to server ownership
    }

    public NetPlayer OnReconnected(AbstractNetPeer peer, BasePlayer existingPawn)
    {
        var player = _manager.AddPlayer(peer);
        _manager.AddController<BasePlayerController>(player, existingPawn);
        return player;
    }
}
```

## How it works

### The possession API

`StartControl(pawn)` binds a pawn to this controller and rewires the pawn's ownership to the controller's owner; it implicitly releases any previously controlled pawn. `StopControl()` detaches the pawn, which reverts to server ownership (or its parent's). `DestroyWithControlledEntity()` kills both at once — it is what `RemovePlayer` calls for the player's controller by default. All three are server-only and silently do nothing on the client.

### The ordering guarantee

`BeforeControlledUpdate()` runs from the pawn's `base.Update()`, immediately before the pawn's own logic — a per-tick guarantee that input handling precedes movement. Do not rely on the relative `Update()` order of the controller and pawn *entities* instead: entity update order follows creation order, which is not a contract. Anything that must happen "just before my pawn moves" belongs in `BeforeControlledUpdate`.

### Finding controllers and players

On the server: `GetPlayerController(NetPlayer | byte | AbstractNetPeer)` returns the player's `HumanControllerLogic`. On the client: `GetPlayerController<T>()` returns the local player's controller or null. In the other direction, a controller resolves its player with `GetAssignedPlayer()`, and `GetControlledEntity<T>()` / the typed `ControlledEntity` property resolve the pawn.

### Reacting to possession changes

Override `OnControlledEntityChanged(prevPawn)` on the controller — it fires on the server at the change and on the owning client when the change syncs in. The argument is the *previous* pawn; the current one is already readable through `ControlledEntity`.

### The reconnect pattern

A controller should be treated as connection-scoped: when a player drops, detach the pawn with `StopControl`, let `RemovePlayer` destroy the controller, and keep the pawn alive in the world. On reconnect, `AddPlayer` produces a fresh `NetPlayer` (with a *new* player id), a fresh controller is created, and `AddController(player, existingPawn)` re-possesses the old body — ownership rewires to the new id automatically.

The corollary is the design rule from the top of this page: nothing that must survive a reconnect may live in the controller. Its SyncVars start clean with every new controller instance; the pawn's state is what persists.

## Behavior details

# [Server](#tab/server)

Possession is entirely server-driven. `RemovePlayer` without a prior `StopControl` destroys the controller *and* its pawn — the right default for permanent leavers, the wrong one for reconnectable games. A pawn without a controller keeps updating as a server-owned entity.

# [Client](#tab/client)

Only the owning client has the controller entity at all (`EntityFlags.OnlyForOwner`). Other clients observe possession indirectly — the pawn's behavior and owner-dependent field visibility change. On the owner, `OnControlledEntityChanged` fires when the possession change arrives with state.

# [Prediction / rollback](#tab/prediction-rollback)

The controlled-pawn reference is marked never-rollback: rollback re-simulation does not rewind who controls what. Input replay during rollback goes through the same `BeforeControlledUpdate` path with historical inputs, as described in [adding-a-player.md](../getting-started/adding-a-player.md).

***

> [!WARNING]
> **Common mistakes**
>
> * Calling `RemovePlayer` without `StopControl` when the pawn should survive — the default path destroys pawn and controller together.
> * Storing score, inventory or per-match stats in the controller — a reconnect creates a fresh controller and that state is gone; persistent state lives on the pawn or a game entity.
> * Ordering logic by the controller entity's `Update()` relative to the pawn's — creation-order scheduling is not a contract; use `BeforeControlledUpdate`.
> * Calling `StartControl`/`StopControl` on the client and wondering why nothing happens — possession is server-only and the calls are silent no-ops.

## Related pages

- [ai-controllers.md](ai-controllers.md)
- [ownership-and-players.md](ownership-and-players.md)
