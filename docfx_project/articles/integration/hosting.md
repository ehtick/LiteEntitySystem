---
description: Running the server - dedicated headless builds, listen-server via localhost, and resetting the world between matches.
---

# Hosting: dedicated and listen-server

The same `ServerEntityManager` covers both deployment models: a headless process players connect to, or a server living inside a player's own client process with the local client connecting over loopback.

## Dedicated server

A headless build runs only the server manager and its game logic — no views, no input polling.

Two practical points. **Frame rate**: cap the process at the tick rate, since rendering nothing at 1000 fps only burns CPU — in Unity that is `Application.targetFrameRate` under `UNITY_SERVER`, elsewhere your own loop's pacing. **Assemblies**: with the [server/client variant pattern](../getting-started/registering-entity-types.md) the build contains no client classes at all, so no engine-view code is compiled or shipped; registration then differs per side by construction rather than by `#if`.

Server-only entity types — `AiControllerLogic` subclasses — are registered here and not on clients, as they are never synchronized.

## Listen-server

For a player-hosted game, run both managers in one process and let the local client connect to `localhost`:

```csharp
// host: start the server, then connect the local client to it
_server.Start(port: 10515);
_client.Connect("localhost", port: 10515);
```

Nothing is special-cased: the local client goes through the same handshake, baseline and input pipeline as a remote one, over a loopback connection with near-zero latency. That uniformity is the point — one code path to maintain and to debug.

The Unity example ships this as its **Host** button: it loads the server scene additively and connects the client to `localhost`. Note the scene detail there — the additive load uses a separate physics scene, so the server's simulation and the client's do not share one engine world.

> [!NOTE]
> Because the host's own client sees near-zero latency, a listen-server hides latency bugs. Test with simulated latency, or against a second machine, before shipping.

## Resetting the world

`Reset()` destroys every entity, clears filters and singletons and stops the internal clock, returning a manager to its pre-first-update state while keeping its registration and transport. Use it between matches or rounds instead of building a new manager.

On the server, players are cleared as well — they must be re-added, so in practice a reset goes together with reconnecting or re-admitting the connected peers. On the client, drop the manager on disconnect and create a fresh one for the next session, as described in [starting a client](../getting-started/starting-a-client.md).

Local singletons receive `Destroy()` during a reset, so anything they hold is released with the world.

> [!WARNING]
> **Common mistakes**
>
> * An uncapped headless loop — the manager only runs ticks at its fixed rate, so extra iterations are pure waste.
> * Sharing engine state between the two managers in a listen-server — with one physics world or one scene graph, the server's simulation and the client's prediction corrupt each other; keep them separated as the example does.
> * Shipping client-side registration in a dedicated build — it is not an error, but it drags view code and its dependencies into the server binary.
> * Reusing a client manager after a reset instead of creating a new one per connection — session state (player id, buffers) belongs to that connection.

## Related pages

- [limits.md](limits.md)
- [transport.md](transport.md)
