---
description: A map of the Unity example project - how to run it as host or dedicated server, and which script demonstrates which concept.
---

# Example project tour

The [Unity example project](https://github.com/RevenantX/LiteEntitySystemUnityExample) is a small top-down 2D shooter — players and bots, a hitscan weapon, predicted projectiles — and every page of this documentation points into it; this page is the map.

## Running it

Open the project in Unity 2021.2+ and play the `Main` scene. The UI offers two buttons:

* **Host** — loads the `Server` scene additively and connects the local client to `localhost`. This is the listen-server pattern: both managers run in one process, the client talks to its own server over the loopback.
* **Connect** — joins a server at the IP in the text field, so a second editor or build can join the host.

For a dedicated server, build with the dedicated-server target: under `UNITY_SERVER` the example caps `Application.targetFrameRate` to the game's tick rate and runs the `Server` scene logic headless.

> [!NOTE]
> Both `ServerLogic` and `ClientLogic` enable LiteNetLib's latency simulation (~50-60 ms) — the demo is laggy on purpose. That is what makes prediction, interpolation and lag compensation visible: disable `SimulateLatency` to compare.

## The scenes

`Main` holds the client: `ClientObject` (`ClientLogic`), the UI (`UiController`), camera and HUD. `Server` holds a single `ServerObject` (`ServerLogic`). The server scene is loaded additively with `LocalPhysicsMode.Physics2D`, giving each scene its own 2D physics world — which is exactly why physics is wrapped in the `UnityPhysicsManager` singleton entity that calls `Simulate()` manually every logic tick instead of letting Unity auto-step it.

## Script map

### Shared (both sides)

| Script | What it demonstrates |
|---|---|
| `NetworkGeneral.cs` | The class-id enum (`GameEntities`) and the tick rate constant — [registering-entity-types.md](registering-entity-types.md). |
| `GamePackets.cs` | Header-byte routing (`PacketType`), the join packet with the type-map hash, the input struct with `MovementKeys` flags. |
| `BasePlayer.cs` | The pawn: interpolated + lag-compensated position, `AlwaysRollback` health, `SyncString` name, `SyncTimer` cooldown, two RPCs, sync-group reactions, predicted projectile spawn. The densest file in the project. |
| `BasePlayerController.cs` | `HumanControllerLogic<TInput, T>`: input polling in `VisualUpdate`, applying it in `BeforeControlledUpdate`, plus distance-based `ToggleSyncGroup` culling on the server — [adding-a-player.md](adding-a-player.md). |
| `SimpleProjectile.cs` | `PredictableEntityLogic`: client-side spawn prediction, per-tick raycast under lag compensation, `UpdateOnClient` for `VisualUpdate` on remote clients. |
| `UnityPhysicsManager.cs` | A `SingletonEntityLogic` owning the per-scene physics world. |
| `WeaponItem.cs`, `GameWeapon.cs` | Minimal entity stubs — the smallest registrable entities. |
| `PlayerProxy.cs` | The view-side MonoBehaviour reading `InterpolatedValue` every frame — [first-synced-entity.md](first-synced-entity.md). |
| `GamePool.cs`, `Extensions.cs`, `UnityLogger.cs` | Non-entity support code: effect pooling, LiteNetLib serialization helpers, the `ILogger` implementation from [installation.md](installation.md). |

### Server

| Script | What it demonstrates |
|---|---|
| `ServerLogic.cs` | Everything from [starting-a-server.md](starting-a-server.md) in production form: manager creation, join flow with hash verification, packet routing — plus 255 AI bots spawned at startup. |
| `ServerBotController.cs` | `AiControllerLogic<BasePlayer>`: a bot drives the same pawn through the same `SetInput` path as human players, registered on the server only. |

### Client

| Script | What it demonstrates |
|---|---|
| `ClientLogic.cs` | Everything from [starting-a-client.md](starting-a-client.md): manager per connection, `SubscribeToConstructed` view wiring, plus a debug overlay of tick/buffer/jitter diagnostics worth reading while tuning. |
| `UiController.cs` | The Host/Connect flow, including the additive server-scene load for listen-server mode. |
| `ClientPlayerView.cs`, `RemotePlayerView.cs` | Local vs remote player views: camera follow for the owner, a health label for others. |
| `ShootEffect.cs`, `HitEffect.cs` | Pooled visual effects triggered from entity RPC handlers — views, not entities. |

## Where to go next

Section 1 ends here — you have a moving, shooting, predicted player and a map of working reference code. The following sections go deeper into each subsystem, in the same order you met them: world structure, synchronization, then the netcode core.

## Related pages

- [adding-a-player.md](adding-a-player.md)

- [lag-compensation.md](lag-compensation.md)
