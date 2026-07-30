---
description: Diagnosing networked behavior - the debug overlay values, frame and SyncVar inspection, and what common symptoms usually mean.
---

# Debugging and diagnostics

Most networking bugs look identical from the outside — "it's laggy", "it teleports" — so the first tool is not a debugger but an overlay of the manager's own numbers, which distinguish a bandwidth problem from a jitter problem from a game-code problem.

## The overlay

Build it once and keep it in every development build; the example project's client draws exactly this.

**DebugOverlay.cs**

```csharp
public string BuildText(ClientEntityManager m) => $@"
Tick: {m.Tick}  ServerTick: {m.ServerTick}
LastProcessed: {m.LastProcessedTick}  StoredCommands: {m.StoredCommands}
Entities: {m.EntitiesCount}  StateSize: {m.StateSize} B
ServerInputBuffer: {m.ServerInputBuffer}
LerpBuffer: {m.LerpBufferCount} ({m.LerpBufferTimeLength:F3} s)
Jitter: {m.NetworkJitter:F3} / avg {m.AverageJitter:F3}
PendingToRemove: {m.PendingToRemoveEntites}";
```

What each value tells you is in [adaptive timing](adaptive-timing.md); the point of showing them together is that their *combination* identifies the problem.

## Frame and field inspection

`GetCurrentFrameDebugInfo(DebugFrameModes)` returns a one-line description of the current update — including, on the client, whether it is a normal frame or a rollback and which replay step is running. The modes filter what you want to see (`Client`, `Server`, `Rollback`, and combinations), so you can log rollbacks only.

`GetEntitySyncVarInfo(entity, printer)` walks an entity's synchronized fields and reports each name and value through your `IEntitySyncVarInfoPrinter` implementation — the fastest way to answer "what does the server think this entity's state is" versus what the client shows. `GetDiagnosticData` fills a dictionary with per-entity size information from the current state, useful when a state grows unexpectedly.

## Reading the symptoms

| Symptom | Usually means |
|---|---|
| Remote entities move in steps | Views read `.Value` instead of `InterpolatedValue`, or the field lacks `SyncFlags.Interpolated` |
| Remote entities stall, then jump | State buffer starving — check `LerpBufferCount` and jitter; loss or bandwidth, not tuning |
| Your own entity rubber-bands | Prediction disagrees with the server: nondeterminism in `Update`, or state changed outside the simulation |
| Effects play twice on corrections | One-shot code in predicted `Update` without `EntityManager.InNormalState` |
| Shots visibly miss where you aimed | Fields missing `SyncFlags.LagCompensated`, or the hit check runs outside input processing |
| A field never updates on other clients | `SyncFlags.OnlyForOwner`, or a [sync group](../sync/sync-groups.md) disabled for that player |
| Nothing updates at all | Missing `[EntityFlags(EntityFlags.Updateable)]`, or the manager's `Update()` is not pumped |
| Client rejected at join | Registration mismatch — the type-map hash differs between builds |
| State size grows over time | Entities not destroyed, or a `SyncVar` written every tick with a slightly different float |

## Making problems reproducible

Reproduce network conditions locally instead of guessing: LiteNetLib can simulate latency and loss, and the example project enables ~50–60 ms of simulated latency by default precisely so prediction and interpolation behave as they will in production. Testing exclusively on a loopback connection hides every class of bug this documentation's netcode section is about.

For prediction bugs specifically, log `GetCurrentFrameDebugInfo(DebugFrameModes.ClientAndRollback)` around the suspect code: seeing the same tick logged several times with different values is the signature of nondeterministic predicted code.

> [!WARNING]
> **Common mistakes**
>
> * Debugging netcode on a loopback connection only — zero latency and zero jitter hide exactly the bugs you are looking for.
> * Reading diagnostics as game state — `StoredCommands`, buffer depths and jitter describe the connection, not the world.
> * Fixing rubber-banding by weakening prediction (moving logic to the server or disabling rollback for a field) before finding the nondeterminism that caused it.
> * Leaving `Logger.LoggerImpl` unassigned — the library reports registration mismatches, missing `base.RegisterRPC` and compensation misses through it, and stays silent without it.

## Related pages

- [adaptive-timing.md](adaptive-timing.md)
- [prediction-and-rollback.md](prediction-and-rollback.md)
