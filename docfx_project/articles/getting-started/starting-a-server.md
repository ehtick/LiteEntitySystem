---
description: Creating a ServerEntityManager over LiteNetLib - constructor parameters, the update pump, adding and removing players, and routing incoming packets.
---

# Starting a server

A game server is a `ServerEntityManager` plus a transport: the manager simulates the world and builds state packets, the transport moves bytes — below with LiteNetLib 2, which the library ships with as the default.

## Minimal example

**GameServer.cs**

```csharp
using LiteEntitySystem;
using LiteEntitySystem.Transport;
using LiteNetLib;

public enum PacketType : byte
{
    EntitySystem,
    Serialized
}

public class GameServer : ILiteNetEventListener
{
    private LiteNetManager _netManager;
    private ServerEntityManager _entityManager;

    public void Start()
    {
        _entityManager = new ServerEntityManager(
            GameTypes.Build(),              // EntityTypesMap, see "Registering entity types"
            (byte)PacketType.EntitySystem,  // header byte of every LES packet
            30,                             // logic ticks per second
            ServerSendRate.EqualToFPS);     // send a state update every tick

        _netManager = new LiteNetManager(this) { AutoRecycle = true };
        _netManager.Start(10515);
    }

    public void Update() // call every frame of your loop
    {
        _netManager.PollEvents();
        _entityManager.Update();
    }

    void ILiteNetEventListener.OnConnectionRequest(LiteConnectionRequest request) =>
        request.AcceptIfKey("MyGame");

    void ILiteNetEventListener.OnPeerConnected(LiteNetPeer peer)
    {
        NetPlayer player = _entityManager.AddPlayer(new LiteNetLibNetPeer(peer, true));
        var pawn = _entityManager.AddEntity<BasePlayer>();
        _entityManager.AddController<BasePlayerController>(player, pawn);
    }

    void ILiteNetEventListener.OnPeerDisconnected(LiteNetPeer peer, DisconnectInfo info)
    {
        if (peer.Tag != null)
            _entityManager.RemovePlayer((LiteNetLibNetPeer)peer.Tag);
    }

    void ILiteNetEventListener.OnNetworkReceive(LiteNetPeer peer, NetPacketReader reader, DeliveryMethod method)
    {
        if ((PacketType)reader.PeekByte() == PacketType.EntitySystem)
            _entityManager.Deserialize((LiteNetLibNetPeer)peer.Tag, reader.GetRemainingBytesSpan());
        // other PacketType values are your own protocol
    }

    void ILiteNetEventListener.OnNetworkError(System.Net.IPEndPoint endPoint, System.Net.Sockets.SocketError error) { }
    void ILiteNetEventListener.OnNetworkReceiveUnconnected(System.Net.IPEndPoint endPoint, NetPacketReader reader, UnconnectedMessageType type) { }
    void ILiteNetEventListener.OnNetworkLatencyUpdate(LiteNetPeer peer, int latency) { }
}
```

## How it works

### The manager constructor

Four decisions are made here. The `EntityTypesMap` fixes the set of entity classes. The header byte is written as the first byte of every packet the manager produces, letting you run your own packets on the same connection and route by `PeekByte`. `framesPerSecond` is the logic tick rate — every entity's `Update()` runs at this frequency. `ServerSendRate` decides how often state is sent relative to ticks: every tick (`EqualToFPS`), every second (`HalfOfFPS`) or every third (`ThirdOfFPS`). An optional fifth parameter, `MaxHistorySize`, sizes the lag-compensation history and defaults to 32 ticks.

### The update pump

`_netManager.PollEvents()` delivers incoming packets, `_entityManager.Update()` advances time: it accumulates real time and runs zero or more logic ticks, sending state on the ticks that match the send rate. Call both every iteration of your loop. Everything — entity updates, RPC handlers, spawns — runs single-threaded inside this call; game logic must stay on this thread for determinism. Async work such as database access is fine, but its results must re-enter the world only inside the logic path.

### Players

`AddPlayer` turns a transport peer into a `NetPlayer`. Wrapping the LiteNetLib peer in `new LiteNetLibNetPeer(peer, true)` stores the adapter in `peer.Tag`, which is how later callbacks find it. `AddPlayer` returns `null` when the player limit is reached — handle that by rejecting the connection. `RemovePlayer` cleans up the player and destroys the controller they own together with its pawn.

The snippet adds the player directly in `OnPeerConnected`. The example project instead waits for a custom join packet carrying the type-map hash and only then calls `AddPlayer` — combine this page with the hash handshake from [registering-entity-types.md](registering-entity-types.md) for production.

### Receiving data

Everything a client sends to the entity system — inputs and requests — goes through `Deserialize`. Check the first byte yourself and pass only your LES packets; the return value distinguishes `Done`, `Error` (malformed data) and `HeaderCheckFailed` (not an LES packet).

### Custom transport

Nothing above is LiteNetLib-specific except the adapter class. To run on your own transport, extend `AbstractNetPeer` — four members: `SendReliableOrdered`, `SendUnreliable`, `GetMaxUnreliablePacketSize`, `TriggerSend` — and pass your implementation wherever `LiteNetLibNetPeer` appears. `LiteNetLibNetPeer` itself is a reference implementation of exactly this — see [transport layer](../integration/transport.md).

## Behavior details

A newly added player automatically receives the full world as an LZ4-compressed reliable baseline on the next send tick, then switches to unreliable diffs. If a player stops confirming states for longer than `PlayerResyncTimeout` (4 seconds by default, public field), the server falls back to sending a fresh baseline. With zero players connected the simulation still runs, but nothing is sent.

> [!WARNING]
> **Common mistakes**
>
> * Feeding every received packet into `Deserialize` without checking the header byte — your own protocol packets produce `HeaderCheckFailed` noise, and worse, LES packets fed to your own parser corrupt your protocol.
> * Ignoring the `null` return of `AddPlayer` — at the player limit the connection stays open but the player is never part of the game.
> * Touching entities or managers from another thread (timers, async callbacks) — the logic contract is single-threaded; marshal results back into the update loop instead.

## Related pages

- [starting-a-client.md](starting-a-client.md)

- [first-synced-entity.md](first-synced-entity.md)
