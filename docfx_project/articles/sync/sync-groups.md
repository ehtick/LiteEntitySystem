---
description: Toggling groups of SyncVars and RPCs per player - distance culling, visibility systems, and what keeps syncing regardless.
---

# SyncGroups: per-player data visibility

A sync group is a tag on SyncVars and RPCs that the server can switch off for one player: the entity still exists for them, but the tagged data stops arriving — the mechanism behind distance culling and "you don't get enemy positions until you can see them".

## When to use this

* Bandwidth: stop sending detailed state of entities far away from a player.
* Anti-cheat: withhold data a player must not know yet — positions of unseen enemies cannot be extracted from packets that were never sent.

## When not to

* Hiding an entity entirely from everyone but its owner — that is `EntityFlags.OnlyForOwner` at [class level](../world/update-flags.md), no per-player logic needed.
* A single owner-private field — [`SyncFlags.OnlyForOwner`](sync-flags.md) is simpler and static.

## Minimal example

**VisibilityController.cs (server logic in the player's controller)**

```csharp
using LiteEntitySystem;

public class VisibilityController : HumanControllerLogic<PlayerInput, MyPlayer>
{
    private const float VisibleRange = 35f;

    public VisibilityController(EntityParams entityParams) : base(entityParams) { }

    protected override void Update()
    {
        base.Update();
        if (!IsServer || ControlledEntity == null)
            return;

        foreach (var other in EntityManager.GetEntities<MyPlayer>())
        {
            bool visible = Distance(ControlledEntity, other) < VisibleRange;
            ServerManager.ToggleSyncGroup(OwnerId, other, SyncGroup.SyncGroup1, visible);
        }
    }

    private static float Distance(MyPlayer a, MyPlayer b) => 0f; // your metric
}
```

**MyPlayer.cs — tagging the data**

```csharp
[SyncVarFlags(SyncFlags.Interpolated | SyncFlags.SyncGroup1)]
public SyncVar<float> X;

protected override void RegisterRPC(ref RPCRegistrator r)
{
    base.RegisterRPC(ref r);
    r.CreateRPCAction(this, OnShoot, ref _shootRpc,
        ExecuteFlags.SendToOther | ExecuteFlags.SyncGroup1);
}

protected override void OnSyncGroupsChanged(SyncGroup enabledGroups)
{
    // client: show or hide the view as data starts or stops arriving
    SetViewActive(enabledGroups.HasFlagFast(SyncGroup.SyncGroup1));
}
```

## How it works

### Tagging and toggling

There are five groups. A `SyncVar` joins one through `SyncFlags.SyncGroup1`…`SyncGroup5`, an RPC through the matching `ExecuteFlags` values. Untagged members are always sent. The server switches a group per (player, entity) pair with `ToggleSyncGroup(playerId, entity, group, enable)` — typically re-evaluated each tick from the player's controller, as above.

Five groups is a deliberate limit: the flags ride along in existing bit fields, and in practice one or two — "detailed state" and maybe "audio-range events" — cover most games.

### What still syncs

Entity creation and destruction always reach every player who can see the entity, regardless of groups. So a culled player still knows the entity exists — they simply stop receiving its tagged data. Untagged fields keep flowing too, which is how you keep a cheap "still alive, roughly here" signal while withholding the precise state.

An entity's owner is never culled from their own data: toggling a group for the player who owns the entity does nothing.

### On the client

When a group turns off, tagged fields stop updating and hold their last received values; RPCs in that group are not delivered. `OnSyncGroupsChanged(enabledGroups)` fires on the entity so the view can react — the example project deactivates the player's GameObject there, so a culled player neither renders at a stale position nor plays sounds.

When the group turns back on, the server resends the current values of that group's fields, so the entity catches up in one state update.

## Behavior details

# [Server](#tab/server)

Group state is stored per player and consulted while composing each packet: disabled fields are skipped, disabled RPCs are dropped for that player. Re-enabling marks the group's fields as changed so they are resent in full.

# [Client](#tab/client)

There is no client-side control — the client only observes through `IsSyncGroupEnabled(group)` and `OnSyncGroupsChanged`. Values of a disabled group are stale by definition: never drive gameplay logic from them, only presentation.

# [Prediction / rollback](#tab/prediction-rollback)

Culling and prediction are orthogonal: a client predicts entities it owns, and it always receives its own entities' data in full. Fields of a culled remote entity simply stop being corrected while the group is off.

***

> [!WARNING]
> **Common mistakes**
>
> * Reading a culled entity's tagged fields as if current — they hold the last value from before the cull; gate view code with `IsSyncGroupEnabled` and never base logic on them.
> * Tagging position but forgetting the related RPCs — shot and footstep calls keep arriving from entities the player cannot see; tag both sides of a feature with the same group.
> * Toggling from client code — `ToggleSyncGroup` exists on `ServerEntityManager` only; visibility is a server decision.
> * Expecting a culled entity to disappear — creation and destruction always sync; hide the view yourself in `OnSyncGroupsChanged`.

## Related pages

- [../netcode/tick-model.md](../netcode/tick-model.md)
- [sync-flags.md](sync-flags.md)
