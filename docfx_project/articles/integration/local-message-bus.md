---
description: Typed in-process pub/sub for decoupling entities from views and systems - subscribing, publishing, and disposing subscriptions.
---

# LocalMessageBus

`LocalMessageBus` is a typed publish/subscribe channel living beside the world as a local singleton: entities announce that something happened, and unrelated code — UI, audio, analytics — reacts without either side knowing about the other. Nothing here crosses the network.

## When to use this

* Decoupling presentation from simulation: an entity publishes "took damage", the HUD and the audio system subscribe independently.
* Cross-cutting reactions where a direct reference would be awkward — the publisher must not need to know its listeners.

## When not to

* Anything another machine must learn about — that is an [RPC](../sync/rpc-basics.md) or synchronized state; the bus is process-local.
* Communication inside one entity, or between an entity and its own view — a direct call or a [change callback](../sync/bind-on-change.md) is simpler and traceable.
* Gameplay state — a message is an event that vanishes after delivery; a joiner or a rollback sees nothing of it.

## Minimal example

**DamageMessage.cs**

```csharp
using LiteEntitySystem.Extensions;

// TSender declares who is allowed to publish this message
public struct DamageMessage : IBusMessage<Fighter>
{
    public byte Amount;
    public byte RemainingHealth;
}
```

**Publishing, from the entity**

```csharp
public void ApplyDamage(byte amount)
{
    Health.Value = amount >= Health.Value ? (byte)0 : (byte)(Health.Value - amount);

    if (IsClient && EntityManager.InNormalState)
        this.GetOrCreateLocalMessageBus()
            .Send(this, new DamageMessage { Amount = amount, RemainingHealth = Health.Value });
}
```

**Subscribing, from a view or system**

```csharp
public class DamageHud : IDisposable
{
    private readonly BusSubscription _subscription;

    public DamageHud(LocalMessageBus bus) =>
        _subscription = bus.Subscribe<Fighter, DamageMessage>(OnDamage);

    private void OnDamage(Fighter sender, DamageMessage msg) { /* flash the health bar */ }

    public void Dispose() => _subscription.Dispose();
}
```

## How it works

### Messages are typed by sender

`IBusMessage<TSender>` binds a message type to the kind of thing that publishes it, so `Subscribe<Fighter, DamageMessage>` gives handlers a correctly typed sender with no casting. Messages are structs — a published message allocates nothing.

### Getting the bus

The bus is an `ILocalSingleton`, so it lives on the manager: register one with `AddLocalSingleton`, or let `entity.GetOrCreateLocalMessageBus()` create it on first use. Since it is per manager, a listen-server process has two independent buses — the server's and the client's — which is what you want: a client-side effect message never reaches server logic.

### Subscriptions are disposable

`Subscribe` returns a `BusSubscription`; disposing it detaches the handler. Tie that to the lifetime of the subscriber — a view's destruction, a scene teardown — or the handler outlives its object and fires into a dead reference. `Destroy()` on the bus itself drops every channel, and a manager [reset](hosting.md) destroys local singletons for you.

### Publishing during prediction

The bus is not part of the simulation: it has no rollback, no history, no replay. A message published from predicted code fires again on every re-simulation, so gate publications with `EntityManager.InNormalState`, exactly as with any other one-shot effect.

Setting `DebugToLog = true` logs every published message with its sender and handler — useful when tracing who reacts to what.

> [!WARNING]
> **Common mistakes**
>
> * Publishing from predicted code without an `InNormalState` gate — every correction replays the message and the effects behind it.
> * Never disposing subscriptions — handlers accumulate across sessions and keep dead objects alive.
> * Expecting messages to reach other machines — the bus is in-process; use RPCs for that.
> * Carrying state in messages — a message that nobody was subscribed for is simply gone; state belongs in SyncVars.

## Related pages

- [limits.md](limits.md)
- [../world/singletons.md](../world/singletons.md)
