---
type: project
project: Hexbane
area: client
domain: [projects]
status: active
created: 2026-09-07
updated: 2026-09-07
verified: 2026-09-07
tags: [hexbane, client, cqrs, di]
sources: ["client:Application/CQRS/README.md"]
---

# CQRS in the client (`Application/CQRS/`)

A small in-process command/query/event bus. Handlers live in the DI container; the dispatcher resolves them per call. See [[client-architecture]] for where it sits in the layer stack.

## Interfaces (`Application/CQRS/Interfaces/`)

| Message | Handler | Dispatcher call |
|---|---|---|
| `ICommand` (marker) | `ICommandHandler<TCommand>` → `Task HandleAsync(cmd)` | `DispatchAsync(cmd)` / sync `Dispatch(cmd)` |
| `ICommand<TResult>` | `ICommandHandler<TCommand, TResult>` → `Task<TResult> HandleAsync(cmd)` | `DispatchAsync<TCommand, TResult>(cmd)` |
| `IQuery<TResult>` (marker) | `IQueryHandler<TQuery, TResult>` → `Task<TResult> HandleAsync(q)` | `QueryAsync<TQuery, TResult>(q)` |
| `IEvent` (`DateTime Timestamp`), base class `Models/BaseEvent` | `IEventHandler<TEvent>` → `Task HandleAsync(evt)` | `PublishAsync(evt)` |

`ICqrsDispatcher` (`Interfaces/ICqrsDispatcher.cs`) exposes `DispatchAsync` (both forms), `QueryAsync`, `PublishAsync`, the synchronous `Dispatch<TCommand>` and a settable `Logger`. The concrete `CqrsDispatcher` (`Bus/CqrsDispatcher.cs:46-59`) additionally has synchronous `Dispatch<TCommand,TResult>` and `Query<TQuery,TResult>`, all implemented as `.GetAwaiter().GetResult()` over the async versions. They are not on the interface.

Result types are inferred from nothing: both type arguments must be written out (`QueryAsync<GetSpellsQuery, GetSpellsResponse>(new GetSpellsQuery())`, `SpellService.cs:29`).

## Registration

`ServiceBootstrapper.ConfigureServices` (`Game/DI/ServiceBootstrapper.cs:120-127`):

```csharp
services.AddCqrs(enableLogging: true, debugMode: false,
    assemblies: new[] { typeof(GetSpellQueryHandler).Assembly });
```

`AddCqrs(this IServiceCollection, bool enableLogging = true, bool debugMode = false, params Assembly[] assemblies)` (`CqrsRegistration.cs:25`):

- scans the given assemblies for every non-abstract class implementing `ICommandHandler<>`, `ICommandHandler<,>`, `IQueryHandler<,>`, `IEventHandler<>` and registers each closed interface → implementation as **transient** (`CqrsRegistration.cs:52-85`). A new handler instance is created per dispatch, so handlers can take constructor dependencies but must not hold state across calls.
- registers `ICqrsLogger` → `CqrsGodotLogger(debugMode)` as singleton when `enableLogging` is true.
- registers `ICqrsDispatcher` → `CqrsDispatcher(serviceProvider)` as singleton. There is no parameterless constructor and no `Register*Handler` method on the dispatcher; handlers can only come from DI. Manual registration means `services.AddTransient<ICommandHandler<MyCommand>, MyCommandHandler>()` before the provider is built.

Everything is in one assembly (`hexbane.csproj`), so the single assembly passed covers Application, Game and Core. `IMatchDataHandler<T>` (the opcode handlers) are **not** CQRS handlers; `RegisterMatchDataHandlers` (`ServiceBootstrapper.cs:134-152`) registers them separately.

Missing handler → `InvalidOperationException("No handler registered for command type …")` after `Logger.LogError` (`CqrsDispatcher.cs:34-39, 72-77, 100-105`). Missing event handlers → warning only, no throw (`CqrsDispatcher.cs:123-127`).

## Usage

Get the dispatcher by constructor injection (services: `CharacterService.cs:28`, `SpellService.cs:19`, `TutorialSession.cs:82`) or from the container (`DIHost.Services.GetService<ICqrsDispatcher>()`, `MatchEntryDataHandler.cs:40`).

Command without result (`Application/Match/Outgoing/Meditate/`):

```csharp
public class MeditateCommand : ICommand
{
    public string MatchId { get; }
    public MeditateCommand(string matchId) => MatchId = matchId;
}

public class MeditateHandler : ICommandHandler<MeditateCommand>
{
    public async Task HandleAsync(MeditateCommand command)
        => await DuelProtocol.Send("meditate");
}
```

Command with result: `CreateCharacterCommand : ICommand<CreateCharacterResponse>` + `CreateCharacterCommandHandler : ICommandHandler<CreateCharacterCommand, CreateCharacterResponse>` (`Application/Modules/Character/Commands/CreateCharacter/`). Others: `AllocateStatPointsCommand`, `SetTutorialCompletedCommand`, `LearnSpellCommand`, `BlockUserCommand : ICommand<bool>`, `TutorialCommand : ICommand<TutorialResponse>`.

Query: `GetSpellQuery : IQuery<GetSpellResponse>` + `GetSpellQueryHandler` (`Application/Modules/Spell/Queries/GetSpell/`). Queries are plain classes; `GetSpellQuery` also keeps a parameterless constructor for serialization.

Synchronous dispatch is used where the caller is a Nakama socket callback, not an async method: `_cqrsDispatcher.Dispatch(new MatchMessageCommand(opcode, bytes, senderId))` (`Application/ArcaneDuel/Normal/MatchManager.cs:128`, `Bot/BotMatchManager.cs:154`) and `Dispatch(new ClientReadyCommand(...))` (`MatchEntryDataHandler.cs:42`, `GameHudScreen.cs:284`). It blocks the calling thread until the handler completes.

Events: the infrastructure (`IEvent`, `BaseEvent`, `IEventHandler`, `PublishAsync`) is complete but **no event or event handler exists in the codebase** (no `BaseEvent` subclass, no `PublishAsync` caller). Game-wide notifications go through the `GameEvents` autoload instead ([[client-architecture]]). If used: `PublishAsync` resolves all `IEventHandler<TEvent>` registrations, starts them all and awaits `Task.WhenAll` (`CqrsDispatcher.cs:121-146`); handlers run concurrently, and the `try/catch` around each only catches exceptions thrown synchronously before the first `await`.

## Logging

`CqrsGodotLogger` (`Logging/CqrsGodotLogger.cs`) prints `[CQRS INFO|WARNING|ERROR|DEBUG] …` via `GD.Print`/`GD.PushWarning`/`GD.PrintErr`; debug lines only when `debugMode`. What is actually logged today:

- missing command/query handler (error), missing event handlers (warning), event-handler exceptions (error), one debug line per event handler executed;
- every per-dispatch debug line in `CqrsDispatcher` is commented out (`CqrsDispatcher.cs:41,43,79,81,96,107,129,147`);
- handler registration prints `[CQRS DEBUG] Registering handler: …` unconditionally through raw `GD.Print` (`CqrsRegistration.cs:67,80-81`), ignoring `debugMode`.

## Conventions

- One folder per command/query: `Application/Modules/<Feature>/{Commands,Queries}/<Name>/<Name>Command.cs` + `<Name>CommandHandler.cs`, or `Application/Match/Outgoing/<Name>/`.
- Commands change state (usually an RPC or a socket send), queries only read; both are immutable data holders.
- Never add a handler by hand to `ServiceBootstrapper`; implementing the interface in the main assembly is the registration.

## Source of truth in code
- `client:Application/CQRS/Interfaces/*.cs` — message and handler contracts
- `client:Application/CQRS/Bus/CqrsDispatcher.cs` — resolution, sync wrappers, error behaviour
- `client:Application/CQRS/CqrsRegistration.cs` — `AddCqrs`, reflection scan, lifetimes
- `client:Application/CQRS/Logging/CqrsGodotLogger.cs` — log format
- `client:Game/DI/ServiceBootstrapper.cs` — the only `AddCqrs` call, `IMatchDataHandler` registration
