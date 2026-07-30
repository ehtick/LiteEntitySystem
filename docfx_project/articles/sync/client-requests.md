---
description: The reliable client-to-server request channel on HumanControllerLogic - sending with a result callback, subscribing on the server, and its limits.
---

# Client to server: requests

Clients have two upward channels: the per-tick input struct, and *requests* — reliable one-off messages sent from a `HumanControllerLogic` with an optional success callback, for the actions that don't belong in input.

## When to use this

* Occasional intentional actions: buy an item, pick a loadout, vote, change team, use a menu.
* Anything the server may accept or reject, where the client needs to know which.

## When not to

* Per-tick movement and firing — that is the [input struct](../getting-started/adding-a-player.md), which is delta-compressed, redundant and prediction-aware; requests are not.
* Server-to-client messaging — that direction is [RPCs](rpc-basics.md).
* High-frequency streams — every request is a reliable packet; a request per tick is the wrong tool.

## Minimal example

**ShopController.cs**

```csharp
using LiteEntitySystem;
using LiteNetLib.Utils;

public struct BuyItemRequest : INetSerializable
{
    public ushort ItemId;
    public byte Count;

    public void Serialize(NetDataWriter writer)
    {
        writer.Put(ItemId);
        writer.Put(Count);
    }

    public void Deserialize(NetDataReader reader)
    {
        ItemId = reader.GetUShort();
        Count = reader.GetByte();
    }
}

// auto-serialized class form: properties with get AND set are required —
// that is what LiteNetLib's NetSerializer reflects over
public class SellItemRequest
{
    public ushort ItemId { get; set; }
    public byte Count { get; set; }
    public string Reason { get; set; }
}

public class ShopController : HumanControllerLogic<PlayerInput, MyPlayer>
{
    public ShopController(EntityParams entityParams) : base(entityParams)
    {
        if (IsServer)
        {
            SubscribeToClientRequestStruct<BuyItemRequest>(OnBuyItem);
            SubscribeToClientRequest<SellItemRequest>(OnSellItem);
        }
    }

    // client side
    public void RequestBuy(ushort itemId, byte count) =>
        SendRequestStruct(new BuyItemRequest { ItemId = itemId, Count = count },
            success => { /* update the shop UI */ });

    public void RequestSell(ushort itemId, byte count) =>
        SendRequest(new SellItemRequest { ItemId = itemId, Count = count, Reason = "manual" },
            success => { /* update the shop UI */ });

    // server side: the bool becomes the client's callback argument
    private bool OnBuyItem(BuyItemRequest request)
    {
        if (!CanAfford(request)) 
            return false;
        GrantItem(request);
        return true;
    }

    private bool OnSellItem(SellItemRequest request)
    {
        // same contract, reflection-serialized payload
        return true;
    }

    private bool CanAfford(BuyItemRequest request) => true;
    private void GrantItem(BuyItemRequest request) { }
}
```

## How it works

### Two payload styles

**Structs with explicit serialization** — `SendRequestStruct<T>` / `SubscribeToClientRequestStruct<T>`, where `T : struct, INetSerializable`. You write `Serialize`/`Deserialize` yourself; no allocation per send, and full control over the bytes.

**Auto-serialized classes** — `SendRequest<T>` / `SubscribeToClientRequest<T>`, where `T : class, new()`. LiteNetLib's `NetSerializer` reflects over the type, so the payload must be declared as **properties with both a getter and a setter** — plain fields are ignored, and a get-only property will not round-trip. Handy for menu-style data with strings, at the cost of an allocation and reflection.

Types nested inside an auto-serialized class need registering first with `RegisterClientCustomType<T>`, either as an `INetSerializable` struct or with explicit write/read delegates.

### The result callback

Both send methods have an overload taking `Action<bool>`. The server handler's return value decides what the client receives: subscribe with a `Func<T, bool>` to answer explicitly, or with an `Action<T>` when the answer is always success. The callback is invoked on the requesting client when the response arrives — treat it as a confirmation, not as data transport; anything the client must *know* afterwards should be state it can read.

### Where subscriptions live

Subscriptions are per-controller instance, made on the server — the controller's constructor is the natural place. Since a controller exists only for its owning player, a handler always knows who sent the request: the controller's owner. There is no way to spoof another player's identity through this channel.

### Reliability and ordering

Requests are sent reliable ordered on their own: none are lost, and they arrive in the order the client sent them. They are processed on the server at the start of a logic tick, before entity updates.

## Behavior details

# [Server](#tab/server)

Pending requests are read at the beginning of the tick, so a handler's changes are visible to every entity update in that same tick. A request addressed to a controller that no longer exists (the player disconnected, the controller was replaced) is dropped. Malformed payloads are logged and ignored rather than throwing.

# [Client](#tab/client)

Only the owning client can send — the controller exists nowhere else. Callbacks are keyed by request id, so several requests may be in flight at once; each callback fires when its own response arrives.

# [Prediction / rollback](#tab/prediction-rollback)

Sending is refused during rollback re-simulation: the call is ignored with a warning, and a callback-taking overload reports failure immediately. Send requests from `VisualUpdate` or from UI code (both run outside re-simulation), never from `Update` of a predicted entity.

***

> [!WARNING]
> **Common mistakes**
>
> * Sending requests from predicted `Update` code — re-simulation would repeat them, so the library ignores them during rollback and your action silently never happens.
> * Using requests for movement or firing — they are reliable and unbuffered, so a packet loss stalls the queue; continuous player intent belongs in the input struct.
> * Treating the `bool` callback as data — it is a yes/no receipt; results the client needs to act on belong in synchronized state or an RPC.
> * Declaring an auto-serialized request class with fields instead of `{ get; set; }` properties — the reflection-based serializer skips them and the payload arrives empty.
> * Subscribing on the client — subscriptions handle incoming requests, which only ever happens on the server; guard with `IsServer`.

## Related pages

- [syncablefield-basics.md](syncablefield-basics.md)
- [rpc-basics.md](rpc-basics.md)
