---
description: AiControllerLogic - server-only bot brains that drive pawns through the same possession path as human players, at zero network cost.
---

# AI controllers and bots

`AiControllerLogic<T>` is the bot counterpart of a human controller: it possesses a pawn and feeds it commands through the same `BeforeControlledUpdate` path, but it exists only on the server — it is never synchronized, never hashed, and clients cannot tell a bot-driven pawn from a human-driven one.

## When to use this

* NPCs and bots that drive a pawn: enemies, training dummies, lobby fillers (the example project spawns 255 of them at startup).
* Reusing one pawn class for humans and bots — the pawn code does not change at all.

## When not to

* Behavior of a world object itself (a turret tracking targets, a patrolling platform) — put the logic in the entity's own `Update()`; a controller adds value only when the *same pawn* can also be driven by something else.
* Match-flow scripting (wave spawners, round logic) — that is a server-side singleton or plain entity, not a possession relationship.

## Minimal example

A bot wandering with the `MyPlayer` pawn from [adding-a-player.md](../getting-started/adding-a-player.md):

**WanderBot.cs (server assembly)**

```csharp
using System;
using LiteEntitySystem;

public class WanderBot : AiControllerLogic<MyPlayer>
{
    private static readonly Random Rng = new();

    private float _direction;
    private float _changeTimer;

    public WanderBot(EntityParams entityParams) : base(entityParams) { }

    protected override void BeforeControlledUpdate()
    {
        _changeTimer -= EntityManager.DeltaTimeF;
        if (_changeTimer <= 0f)
        {
            _direction = (float)(Rng.NextDouble() * Math.Tau);
            _changeTimer = 2f;
        }
        ControlledEntity?.SetMove(MathF.Cos(_direction), MathF.Sin(_direction));
    }
}
```

**On the server**

```csharp
var pawn = _entityManager.AddEntity<MyPlayer>();
_entityManager.AddAIController<WanderBot>(bot => bot.StartControl(pawn));
```

## How it works

### Server-only by design

AI controllers are excluded from everything network-related: their field changes are never serialized, their RPCs are never enqueued, they are skipped by the registration hash, and on destruction they are removed immediately instead of waiting for client acknowledgements. That is why the client may simply not register them — the example project's client registers every type except `BotController`.

### Plain fields, not SyncVars

Because nothing syncs, bot state is ordinary C# fields — `_direction` and `_changeTimer` above. Anything about the bot that players *should* see (its name tag, its health) belongs on the pawn, which is a normal synced entity owned by the server.

### The same possession, without a player

`AddAIController<T>()` spawns the controller; `StartControl`, `StopControl` and `DestroyWithControlledEntity` work exactly as on [controllers-in-depth.md](controllers-in-depth.md). There is no `NetPlayer` behind a bot: the pawn stays server-owned, and player-based lookups (`GetPlayerController`) will never return an AI controller. `IsBot` distinguishes the two branches when code holds a base `ControllerLogic` reference.

### Randomness is fine here

`BeforeControlledUpdate` of an AI controller runs only on the server, so nondeterministic randomness is harmless — nothing re-simulates it. This is an exception, not the rule: the same `Random` calls inside a *predicted* pawn or projectile would desync prediction, since rollback re-runs that code expecting identical results.

## Behavior details

# [Server](#tab/server)

The controller is `Updateable`: `BeforeControlledUpdate` runs every tick right before its pawn's `Update`, same as for humans. Destruction is immediate — no destroy event is sent, because clients never knew the controller existed.

# [Client](#tab/client)

Nothing exists client-side. Bot pawns arrive as ordinary server-owned entities — interpolated, lag-compensated, damageable — indistinguishable from any other remote pawn unless your game data (a name, a flag on the pawn) says otherwise.

# [Prediction / rollback](#tab/prediction-rollback)

Bots are never predicted — no client owns them. Their pawns still participate in other clients' prediction normally: an `AlwaysRollback` health field on a bot's pawn predicts damage the same way it does on a human's pawn.

***

> [!WARNING]
> **Common mistakes**
>
> * Declaring SyncVars or RPCs in an AI controller and expecting clients to see them — nothing in an AI controller ever syncs; player-visible state belongs on the pawn.
> * Reaching a bot through player-based APIs — there is no `NetPlayer`, so `GetPlayerController` and connection-oriented flows skip bots entirely; keep your own registry of bot controllers if you need one.
> * Copying bot code with `Random` into a predicted entity — safe in a server-only controller, a desync source anywhere rollback re-simulates.

## Related pages

- [controllers-in-depth.md](controllers-in-depth.md)
- [update-flags.md](update-flags.md)
