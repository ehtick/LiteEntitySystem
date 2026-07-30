---
description: Fixed logic ticks - tick rate and send rate, delta time, the wrapping tick counter, and how the manager turns real time into ticks.
---

# The tick model

Game logic advances in fixed steps called ticks: the server picks the rate, both sides run identical step sizes, and every prediction, rollback and lag-compensation mechanism is counted in ticks rather than seconds.

## Choosing the rates

Two numbers are set when the server starts: how often logic runs, and how often state is sent.

`framesPerSecond` — the logic tick rate. Higher means finer simulation, lower input latency and less time between a hit happening and everyone learning about it, at proportionally more CPU on the server.

* **60** — the modern standard for competitive shooters, and what large titles have run at for years. Kills register faster and the "shot me from behind a wall" cases shrink, because each client's view of the world is less stale.
* **30** — a solid default for action games, and the practical ceiling on mobile.
* **15** — enough for slow-paced gameplay: strategy, turn-flavored or puzzle games.

There is no universally right value, only that trade-off against server load — for a competitive shooter, spend it on 60.

`ServerSendRate` — how many ticks pass between state updates: `EqualToFPS` (every tick), `HalfOfFPS`, `ThirdOfFPS`. Lowering it cuts bandwidth without changing simulation quality; the cost is a longer interpolation delay for remote entities, since clients receive fewer snapshots to blend between.

The client is told both values in the baseline, so `Tickrate`, `DeltaTime` and `ServerSendRate` are read-only there.

## Time in entity code

**Cave.cs**

```csharp
protected override void Update()
{
    // fixed logic tick: the same value on server and client, every tick
    _timer += EntityManager.DeltaTimeF;

    if (_timer >= 5f)
    {
        _timer = 0f;
        Collapse();
    }
}
```

`DeltaTimeF` (float) and `DeltaTime` (double) are exactly `1 / Tickrate` — constant, identical on both sides, and the only correct time source inside `Update`. Never use engine frame time there: `Time.deltaTime` varies per frame and would make prediction diverge from the server. For per-frame view code, `VisualDeltaTimeF` gives the render-frame delta, and `LerpFactor` reports how far the current frame sits between two logic ticks.

## The tick counter

`EntityManager.Tick` is a `ushort` that wraps around at 65535 — about 18 minutes at 60 ticks per second, 36 at 30. Comparing tick numbers with `<` or `>` breaks at that boundary, so all comparisons go through `Utils.SequenceDiff`, which returns the signed distance accounting for wraparound:

```csharp
if (Utils.SequenceDiff(deadlineTick, EntityManager.Tick) <= 0)
    OnDeadlineReached();
```

The client's `Tick` is its own counter, starting from zero at connection — it is not comparable to server tick numbers at all. The server's timeline as seen by the client is `ServerTick`. Both wrap, and both need `SequenceDiff`.

## How the manager advances time

`Update()` on the manager is the pump. It measures real time since the previous call, accumulates it, and runs as many logic ticks as fit — zero if the frame was short, several if the frame was long — then leaves the remainder for the next call. Rendering is untouched by this: you call `Update()` once per frame, and the fixed steps happen inside.

Two consequences worth knowing. A long stall (loading, breakpoint) does not replay the missing seconds tick by tick — the manager caps how many ticks one call may run and drops the rest. And on the client the effective step length is nudged slightly by the adaptive clock described in [adaptive timing](adaptive-timing.md), which keeps the state buffer and the server's input buffer healthy.

> [!WARNING]
> **Common mistakes**
>
> * Using engine frame time inside `Update` — the step must be `DeltaTimeF`, or client and server compute different results and prediction fights the server.
> * Comparing ticks with `<`/`>` — correct until the counter wraps, then subtly broken; use `Utils.SequenceDiff`.
> * Comparing the client's `Tick` to a server tick value — different origins; use `ServerTick` for the server timeline.
> * Assuming a fixed number of ticks between a server change and its arrival — the send rate sets the sampling, but latency and jitter set the delay.

## Related pages

- [update-vs-visualupdate.md](update-vs-visualupdate.md)
- [adaptive-timing.md](adaptive-timing.md)
