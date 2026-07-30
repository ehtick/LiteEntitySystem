---
description: How to add LiteEntitySystem to a .NET, Godot, MonoGame or Unity project - via NuGet or by copying sources - and hook up logging.
---

# Installation and project setup

LiteEntitySystem targets .NET Standard 2.1 and installs either as a NuGet package (works for almost every engine) or as a copy of the sources plus the Roslyn analyzer DLL.

Whichever path you take, install the analyzer with it. `SyncVar<T>` fields must only be modified through `.Value` — assigning a whole new struct (`_health = new SyncVar<byte>()`) silently breaks synchronization, and `LiteEntitySystemAnalyzer` turns that mistake into a compile-time error.

## Option 1: NuGet package

For pure .NET servers, Godot, MonoGame and other NuGet-capable environments:

**shell**

```
dotnet add package LiteEntitySystem
```

The package brings everything with it: the dependencies (LiteNetLib 2.x, K4os.Compression.LZ4), the internal `RefMagic.dll`, and the Roslyn analyzer — no separate analyzer setup needed. In Unity, NuGet also works through a NuGet-for-Unity tooling package if you prefer it over copying sources.

## Option 2: copy the sources

Copy two things from the [repository](https://github.com/RevenantX/LiteEntitySystem):

* the `LiteEntitySystem/LiteEntitySystem` folder (includes `ILPart/RefMagic.dll` and an `.asmdef` for Unity);
* `AnalyzerBinary/LiteEntitySystemAnalyzer.dll`.

With this path the dependencies are yours to provide: LiteNetLib 2.x and K4os.Compression.LZ4 (plus `System.Runtime.CompilerServices.Unsafe` where the target framework doesn't ship it).

## Unity setup

The [example project](https://github.com/RevenantX/LiteEntitySystemUnityExample) is the reference layout for the sources path. Requirements: Unity 2021.2 or later; IL2CPP is supported.

* `Assets/Plugins/LiteEntitySystem/` — the library sources with their `.asmdef`.
* `Assets/Plugins/LiteEntitySystemAnalyzer.dll` — imported with the asset label `RoslynAnalyzer`, which is what makes Unity run it as an analyzer.
* `Assets/Plugins/Dependencies/` — `K4os.Compression.LZ4.dll` and `System.Runtime.CompilerServices.Unsafe.dll`.
* LiteNetLib — as a UPM package `com.revenantx.litenetlib` from the OpenUPM scoped registry (see the example's `Packages/manifest.json`); the library's `.asmdef` references the `LiteNetLib` assembly by that name.

## Hook up logging

The library logs through a pluggable `ILogger` and stays silent until you assign one. Do this once at startup, before creating any manager — most setup mistakes (unregistered types, missing base calls) are reported here.

**UnityLogger.cs**

```csharp
using LiteEntitySystem;

public class UnityLogger : ILogger
{
    public void Log(string log) => UnityEngine.Debug.Log(log);
    public void LogWarning(string log) => UnityEngine.Debug.LogWarning(log);
    public void LogError(string log) => UnityEngine.Debug.LogError(log);
}
```

**Startup**

```csharp
LiteEntitySystem.Logger.LoggerImpl = new UnityLogger();
```

Outside Unity, implement the same three methods over `Console` or your engine's log.

> [!WARNING]
> **Common mistakes**
>
> * Skipping the analyzer — `x = new SyncVar<T>()` compiles without it and breaks synchronization with no error at runtime.
> * In Unity, dropping `LiteEntitySystemAnalyzer.dll` into the project without the `RoslynAnalyzer` label — Unity then treats it as a plain plugin and the analyzer never runs.
> * Not assigning `Logger.LoggerImpl` — the library swallows all warnings and errors, and real problems (type registration mismatches, missing `base.RegisterRPC`) go unnoticed.

## Related pages

- [registering-entity-types.md](registering-entity-types.md)

- [starting-a-server.md](starting-a-server.md)
