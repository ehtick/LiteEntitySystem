---
description: The entity parent/child tree - SetParent, Childs, ownership cascade, destruction cascade, change hooks, and how views should mirror it.
---

# Parent and child entities

Entities form a synchronized logical tree: a weapon hangs on a player, an item sits in an inventory, a projectile belongs to its shooter — and attachment carries consequences, because ownership flows down the tree and destruction cascades through it.

## When to use this

* Attachment relationships with gameplay meaning: equipment on a pawn, cargo in a vehicle, area triggers tied to their zone.
* Anything that must transfer ownership together with possession — a weapon handed to another player starts being predicted by *that* player automatically.

## When not to

* Pure visual nesting (a hat bone, muzzle flash under a barrel) — that is the view's transform hierarchy, managed in engine code; the entity tree is not required for it.
* References without attachment semantics — a turret's current target is a `SyncVar<EntitySharedReference>`, not a child.

## Minimal example

**On the server**

```csharp
// spawn attached: inherits the pawn's owner immediately
var weapon = _entityManager.AddEntity<GameWeapon>(playerPawn);

// hand it to another player — ownership (and prediction) follow
weapon.SetParent(otherPawn);

// drop it into the world — reverts to server ownership
weapon.SetParent(null);
```

**GameWeapon.cs**

```csharp
using LiteEntitySystem;

public class GameWeapon : EntityLogic
{
    public GameWeapon(EntityParams entityParams) : base(entityParams) { }

    protected override void OnParentChanged(EntityLogic oldParent)
    {
        // fires on server and on every client, ordered with state:
        // re-attach the view to the new parent's view here
    }

    protected override void OnBeforeParentDestroy()
    {
        // parent is about to die; reparent here to survive the cascade
        if (IsServer)
            SetParent(null);
    }
}
```

## How it works

### The tree

`Parent`/`ParentId`/`GetParent<T>()` walk up; `Childs` — a synchronized set of `EntitySharedReference` — walks down (`Count`, `Contains`, enumeration, `ToArray`). `SetParent` is server-only and a silent no-op on the client; spawning with `AddEntity<T>(parent, …)` attaches at creation, which is cheaper than spawn-then-reparent.

### Ownership flows down

Attaching an entity rewrites its owner to the parent's owner, recursively through its whole subtree — and with ownership comes everything from [ownership-and-players.md](ownership-and-players.md): who predicts it, who receives owner-only fields. Detaching (`SetParent(null)`) reverts the subtree to server ownership. This is the same mechanism possession uses, so a weapon on a pawn is predicted by the pawn's player with no extra wiring.

### Change hooks

Three notifications, firing on the server and on every client in order with state: `OnParentChanged(oldParent)` on the moved entity, `OnChildAdded(child)` / `OnChildRemoved(child)` on the parents. The arguments are the *previous* parent and the affected child; current relations are already readable through the properties.

### The destruction cascade

Destroying a parent first gives every child `OnBeforeParentDestroy()`, then destroys the children that are still attached. The hook is the escape hatch: a child that reparents itself there (as `GameWeapon` above) leaves `Childs` and survives the cascade. `OnBeforeParentDestroy` runs where the destruction is simulated — server, or owner client for local entities.

### Views mirror the tree, not the other way around

`Childs` *can* drive an engine transform hierarchy, but the recommended split is: the entity tree is the logical truth, and views react to it — attach and detach engine objects in `OnParentChanged`/`OnConstructed` on the client. The library never touches transforms; if nothing re-attaches the view, the weapon's model stays where it was while its entity moves to a new parent.

## Behavior details

# [Server](#tab/server)

`SetParent` applies immediately: `Childs` sets update, ownership cascades, hooks fire, and the change ships to clients in order with state. Destruction cascades run here (or on the owning client for predicted local entities).

# [Client](#tab/client)

The tree is read-only: `SetParent` is a no-op, `Childs` and `Parent` reflect the last applied state, hooks fire as the changes sync in. Reparenting that shifts ownership to or from the local player automatically starts or stops prediction for that subtree.

# [Prediction / rollback](#tab/prediction-rollback)

`Childs` participates in rollback with custom logic: predicted membership changes (a projectile predictively added under its shooter) are reverted to the server's set and re-applied during re-simulation. This is internal to `AddPredictedEntity` — game code doesn't manage it.

***

> [!WARNING]
> **Common mistakes**
>
> * Expecting the view to move when the entity reparents — the library moves no transforms; re-attach views in `OnParentChanged` (or manage visual nesting entirely view-side).
> * Attaching an entity to a pawn and forgetting what ownership now means — the pawn's player starts predicting it and receives its owner-only fields; detach reverts to server ownership.
> * Destroying a parent and assuming children survive — they die with it unless they reparent inside `OnBeforeParentDestroy`.
> * Calling `SetParent` on the client and wondering why nothing happened — the tree is server-authoritative; the call is a silent no-op.

## Related pages

- [finding-entities.md](finding-entities.md)
- [ownership-and-players.md](ownership-and-players.md)
