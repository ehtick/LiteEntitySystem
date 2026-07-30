---
description: Creating a ClientEntityManager over LiteNetLib - the connection lifecycle, receiving the baseline, and reacting to entities as they appear.
---

# Starting a client

The client mirrors the server setup with one structural difference: a `ClientEntityManager` is bound to a single connection, so it is created when the connection is established — and the tick rate is not configured here at all, it arrives from the server with the first world state.

## Minimal example

**GameClient.cs**

```csharp
using LiteEntitySystem;
using LiteEntitySystem.Transport;
using LiteNetLib;

public class GameClient : ILiteNetEventListener
{
    private LiteNetManager _netManager;
    private ClientEntityManager _entityManager;

    public void Start()
    {
        _netManager = new LiteNetManager(this) { AutoRecycle = true };
        _netManager.Start();
        _netManager.Connect("localhost", 10515, "MyGame");
    }

    public void Update() // call every frame of your loop
    {
        _netManager.PollEvents();
        _entityManager?.Update();
    }

    void ILiteNetEventListener.OnPeerConnected(LiteNetPeer peer)
    {
        _entityManager = new ClientEntityManager(
            GameTypes.Build(),                 // same registration as the server
            new LiteNetLibNetPeer(peer, true), // this connection
            (byte)PacketType.EntitySystem);    // same header byte as the server

        // optional: react to spawns from outside the entity (camera, UI, sounds).
        // The entity's own view is usually created in its OnConstructed instead.
        _entityManager.GetEntities<BasePlayer>().SubscribeToConstructed(
            player => { /* e.g. attach the camera if player.IsLocalControlled */ },
            callOnExisting: true);
    }

    void ILiteNetEventListener.OnPeerDisconnected(LiteNetPeer peer, DisconnectInfo info)
    {
        _entityManager.Reset(); // cleanup resources - all entities got destroy
        _entityManager = null; // a manager is per-connection; make a new one on reconnect
    }

    void ILiteNetEventListener.OnNetworkReceive(LiteNetPeer peer, NetPacketReader reader, DeliveryMethod method)
    {
        if ((PacketType)reader.PeekByte() == PacketType.EntitySystem)
            _entityManager.Deserialize(reader.GetRemainingBytesSpan());
    }

    void ILiteNetEventListener.OnConnectionRequest(LiteConnectionRequest request) =>
        request.Reject(); // clients don't accept connections

    void ILiteNetEventListener.OnNetworkError(System.Net.IPEndPoint endPoint, System.Net.Sockets.SocketError error) { }
    void ILiteNetEventListener.OnNetworkReceiveUnconnected(System.Net.IPEndPoint endPoint, NetPacketReader reader, UnconnectedMessageType type) { }
    void ILiteNetEventListener.OnNetworkLatencyUpdate(LiteNetPeer peer, int latency) { }
}
```

## How it works

### The manager constructor

Three arguments instead of the server's four: the same `EntityTypesMap` registration, the peer of this connection wrapped in `LiteNetLibNetPeer`, and the same header byte the server uses. There is no tick-rate parameter — `Tickrate` and the server's send rate arrive with the baseline, so the client always runs at whatever the server decides.

### The connection lifecycle

The manager is created in `OnPeerConnected`, because it needs the live peer. On disconnect, drop the reference and create a fresh manager on the next connection — a manager instance represents one session with one server world. The `?.` in the update pump covers the window when no connection exists.

As on the server page, the snippet skips the production join flow: the example project sends a join packet with the type-map hash right after connecting and the server admits the player only after verifying it — see [registering-entity-types.md](registering-entity-types.md).

### Reacting to entities

At connect time the world is empty; entities appear when the baseline is applied and later as the server spawns them. There are two complementary places to react, and the example project uses both.

The normal practice for an entity's own view is the entity itself: override `OnConstructed` and, on the client, create the engine objects there (the example's `BasePlayer` builds its GameObject in `OnConstructed` and destroys it in `OnDestroy`). The entity owns its view for its whole lifetime, and no external wiring is needed.

`GetEntities<T>().SubscribeToConstructed(callback, callOnExisting: true)` serves reactions that live *outside* the entity — binding the camera to the local player, updating UI rosters, playing a join sound. It fires for every constructed entity of the type, including ones that already exist at subscription time, and `IsLocalControlled` distinguishes the local player from remote ones.

### Receiving data

Route by the header byte exactly like the server, and hand LES packets to `Deserialize` — here without a player argument, since the client has only one counterpart.

## Behavior details

Until the first baseline arrives, the manager is dormant: `Tickrate` is zero and `Update()` returns immediately. The baseline then sets the tick and send rates, assigns your `PlayerId`, constructs every visible entity and runs their construction callbacks.

From that point the client ticks on its own clock and starts sending input packets every tick (once a controller exists — see [adding-a-player.md](adding-a-player.md)). `Tick` is the client's own counter: it starts from zero at connection, so its value has no relation to the server's tick numbers — `ServerTick` is the interpolated server timeline, and it is the one to use when reasoning about server time.

The client's clock is also elastic. To play the incoming simulation smoothly it speeds up or slows down by fractions of a second: slowing keeps the queue of received states from draining to empty, speeding up plays buffered states faster to catch up with the server and shrink the buffer. The same mechanism nudges the prediction clock so the server always has a small reserve of the client's inputs. This is automatic; it only becomes visible when tuning buffers or reading diagnostics. If the connection degrades badly, the server resends a baseline and the client resynchronizes without any action from your code.

> [!WARNING]
> **Common mistakes**
>
> * Expecting entities right after `Connect` — the world exists only after the baseline is applied. Subscribe with `SubscribeToConstructed` instead of checking `GetEntities` counts in a loop.
> * Reusing the old `ClientEntityManager` after a reconnect — it belongs to the dead session; create a new one per connection.
> * Pumping `_entityManager.Update()` without the null check — one frame between disconnect and reconnect is enough for a `NullReferenceException`.

## Related pages

- [first-synced-entity.md](first-synced-entity.md)

- [adding-a-player.md](adding-a-player.md)
