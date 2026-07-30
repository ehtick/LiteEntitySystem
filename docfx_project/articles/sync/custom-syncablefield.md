---
description: Building your own SyncableField - registering client actions, sending changes, restoring full state, and adding rollback support.
---

# Writing a custom SyncableField

When no built-in type fits, a custom `SyncableField` gives you a synchronized object of your own: you decide what a change is, send it as an internal RPC, and re-send the whole contents when a new client needs it.

## When to use this

* A data structure the [built-ins](built-in-syncable-fields.md) don't cover, where sending only the edits matters.
* Domain objects with their own invariants — a deck of cards, a build grid, a cooldown table.

## When not to

* A fixed-size unmanaged value — [`SyncVar<T>`](syncvar.md).
* Something a built-in already does; wrapping a `SyncList<T>` in your own entity API is simpler and already rollback-aware.

## Minimal example

A flag grid where any cell can be flipped:

**SyncFlagGrid.cs**

```csharp
using LiteEntitySystem;

public class SyncFlagGrid : SyncableField
{
    private struct CellChange
    {
        public ushort Index;
        public bool Value;
    }

    private readonly bool[] _cells;

    private static RemoteCall<CellChange> _setCellAction;
    private static RemoteCallSpan<bool> _initAction;

    public SyncFlagGrid(int size) => _cells = new bool[size];

    public bool this[int index]
    {
        get => _cells[index];
        set
        {
            if (_cells[index] == value)
                return;
            _cells[index] = value;
            ExecuteRPC(_setCellAction, new CellChange { Index = (ushort)index, Value = value });
        }
    }

    protected internal override void RegisterRPC(ref SyncableRPCRegistrator r)
    {
        r.CreateClientAction(this, OnSetCell, ref _setCellAction);
        r.CreateClientAction(this, OnInit, ref _initAction);
    }

    protected internal override void OnSyncRequested() => ExecuteRPC(_initAction, _cells);

    private void OnSetCell(CellChange change) => _cells[change.Index] = change.Value;

    private void OnInit(ReadOnlySpan<bool> data) => data.CopyTo(_cells);
}
```

## How it works

### Client actions instead of RPCs

Inside a syncable field the registrator is `SyncableRPCRegistrator` and the method is `CreateClientAction` — the same static-handle pattern as [entity RPCs](rpc-basics.md), minus the flags. You never pass `ExecuteFlags` here: the field's own `[SyncVarFlags]` visibility is translated into the delivery rule for every call it sends — no flag means all players, `OnlyForOwner` means the owner alone, `OnlyForOtherPlayers` means everyone else, and sync-group tags route the calls like any other RPC. One rule for the whole field, fixed when it is bound to the entity.

A client action always executes on the receiving client. `RegisterRPC` here does not need a `base` call unless your own base class registers something.

### Sending changes

`ExecuteRPC` from inside the field sends to the players the field's visibility allows. It only does anything on the server, so mutating methods can be written without side checks — on a client they change local memory and nothing else. Compare before sending, as the setter above does: an unchanged value should not produce traffic.

### Restoring full state

`OnSyncRequested` is called when a client needs the entity's complete state — on join, and whenever the server must resend it. Send everything in one call, typically with a span action. Without this override, a client that connects after the edits sees an empty field: the incremental calls it missed are never replayed.

### Choosing the payload shape

Design one small struct per kind of change (`CellChange` above), and one bulk action for `OnSyncRequested`. Payload rules are the [RPC payload rules](rpc-payloads.md): unmanaged values, spans valid only inside the handler, `ISpanSerializable` when the shape is variable.

## Adding rollback support

Derive from `SyncableFieldCustomRollback` when clients may modify the field predictively. Four members carry the contract:

| Member | Role |
|---|---|
| `BeforeReadRPC()` | Incoming server data is about to be applied — redirect writes to your server-side copy |
| `AfterReadRPC()` | Server data applied — restore the working copy from it |
| `OnRollback()` | Reset the working copy to the server-confirmed contents |
| `MarkAsChanged()` | Call from mutators so the entity joins the client's rollback set |

The pattern is to keep two copies: the confirmed server state and the working state predicted code sees. The library's own `Childs` field is the reference implementation.

## Behavior details

# [Server](#tab/server)

Mutations produce client actions immediately, ordered with state. `OnSyncRequested` runs per client that needs a full copy — keep it allocation-free if entities are numerous.

# [Client](#tab/client)

Incoming actions are applied in order. Local mutations of a plain syncable field affect only local memory; nothing is sent upward, and nothing reconciles them with the server.

# [Prediction / rollback](#tab/prediction-rollback)

A plain `SyncableField` is invisible to rollback — predicted edits are never reverted. Only `SyncableFieldCustomRollback` types take part, through the four members above.

***

> [!WARNING]
> **Common mistakes**
>
> * No `OnSyncRequested` — the field looks empty to every client that joins after its contents were set.
> * Non-static action handles — registration is per class; instance handles waste memory and break the registration model.
> * Sending on every write without comparing — a value reassigned each tick becomes traffic each tick.
> * Predictive edits on a plain `SyncableField` — without `SyncableFieldCustomRollback` they are never reset, and the client drifts away from the server.

## Related pages

- [sync-groups.md](sync-groups.md)
- [built-in-syncable-fields.md](built-in-syncable-fields.md)
