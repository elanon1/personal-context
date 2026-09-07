---
type: project
project: Hexbane
area: audits
domain: [projects]
status: active
created: 2026-09-07
updated: 2026-09-07
tags: [hexbane, audit, docs-verification]
---

> Audit report from the 2026-09-07 documentation consolidation (see [[10_Projects/hexbane/_state|_state]] decisions log). Discrepancies here were already applied to the merged notes; the "code smells" and "open questions" sections are the backlog. Paths: `client:` = `~/RiderProjects/hexbane`, `server:` = `~/GolandProjects/hexbane-server`.

# Report: code-folder READMEs (client)

## 1. Input → verdict

| Input | Verdict |
|---|---|
| `client:Application/CQRS/README.md` (221 lines) | merged into `staging/client/cqrs.md` (rewritten; examples replaced with real ones from the repo, non-compiling API dropped) |
| `client:Application/ArcaneDuel/README.md` (169 lines, "Spell Effect System") | merged into `staging/client/spell-effect-system.md` (architecture/API only; asset mapping stays in `spell-vfx-configuration.md`) |
| `client:Game/README.md` (128 lines, Polish, GDScript→C# migration) | dropped: stale (describes a DI/autoload layout that no longer exists, see §5); the few surviving facts are listed in §4 for `architecture.md` |

## 2. Discrepancies found

### Application/CQRS/README.md

| Doc said | Code says | Evidence |
|---|---|---|
| `services.AddCqrs(Assembly.GetExecutingAssembly())` | first parameter is `bool enableLogging`; passing an `Assembly` positionally does not compile | `Application/CQRS/CqrsRegistration.cs:25` |
| Manual registration: `new CqrsDispatcher()` then `dispatcher.RegisterCommandHandler<…>()`, `RegisterQueryHandler<…>()`, `RegisterEventHandler<…>()` | only constructor is `CqrsDispatcher(IServiceProvider)`; no `Register*Handler` method exists; handlers are resolved from DI per call | `Application/CQRS/Bus/CqrsDispatcher.cs:20-23, 32, 70, 99, 121` |
| "Multiple Event Handlers" via `RegisterEventHandler` | same; multiple handlers = multiple DI registrations, resolved with `GetServices<IEventHandler<T>>()` | `CqrsDispatcher.cs:121` |
| Dispatcher API is async only | concrete class also has sync `Dispatch<TCommand>`, `Dispatch<TCommand,TResult>`, `Query<TQuery,TResult>`; only `Dispatch<TCommand>` is on the interface; sync form is what match managers actually use | `CqrsDispatcher.cs:46-59`, `Interfaces/ICqrsDispatcher.cs:52`, `Application/ArcaneDuel/Normal/MatchManager.cs:128` |
| Logging reports "handler registration, command/query dispatch, event publishing" | all per-dispatch debug lines are commented out; registration logs go through raw `GD.Print` regardless of `debugMode` | `CqrsDispatcher.cs:41,43,79,81,96,107,129,147`; `CqrsRegistration.cs:67,80-81` |
| Events (`BaseEvent`, `IEventHandler`) presented as a used feature | zero events, zero event handlers, zero `PublishAsync` callers in the repo | `grep -rn 'BaseEvent\|IEventHandler<\|PublishAsync(' Application Game Core` → only `Application/CQRS/` |
| `PublishAsync` "all handlers will be executed" (implies sequential) | handlers started together and awaited with `Task.WhenAll`; the try/catch only sees synchronous throws | `CqrsDispatcher.cs:130-146` |
| Handler lifetime unspecified | transient | `CqrsRegistration.cs:82` |

### Application/ArcaneDuel/README.md

| Doc said | Code says | Evidence |
|---|---|---|
| Application layer lives in `Application/ArcaneDuel/` (`SpellEffectFactory`, `SpellEffectManager`, `SpellEffectConfigurations`) | that folder holds only the two match managers; the effect classes are in `Application/Modules/Spell/Effects/` | `find Application/ArcaneDuel -name '*.cs'` → `Bot/BotMatchManager.cs`, `Normal/MatchManager.cs` |
| Game layer: `Game/ScenesV2/Game/ArcaneDuel/Components/SpellAnimationController` | `Game/ScenesV2` does not exist; controller is at `Game/ScenesV3/GameHud/ArcaneDuel/Components/SpellAnimationController.cs` | `ls Game/ScenesV2` → no such directory |
| `SpellEffectConfiguration` has `AudioPath` | no such property (fields: SpellId, EffectType, Direction, EffectScenePath, Duration, Parameters, AutoCleanup, RenderPriority) | `Core/Spells/SpellEffectRegistry.cs:53-78` |
| "The system automatically registers all predefined spells" | 10 of 20 `Create*()` entries are commented out of `GetDefaultConfigurations()` (great_heal, reflection, arcane_shield, fireball, explosion, paralyze, venom_shot, aqua_pulse, ember_burst, frost_cut) | `Application/Modules/Spell/Effects/SpellEffectConfigurations.cs:16-38` |
| "Auto-Cleanup: effects automatically remove themselves after duration" | the cleanup timer is created but its `Timeout` handler is commented out; nothing is stopped by it | `SpellEffectFactory.cs:135-148` (line 140) |
| "Event-Driven: uses game events for spell activation" (opcode messages) | current driver is `GameEvents.OnDuelEvent` / `OnDuelSnapshot` (duel v2); the message-based handlers `OnSpellCasting/OnSpellAccepted/OnGameEffectRemoved` have no callers | `SpellEffectManager.cs:54-61, 75-197`; no callers found by grep |
| Four effect types incl. AreaOfEffect and Beam | no `IAreaOfEffectEffect` / `IBeamEffect` implementation and no config uses those types | `grep 'public partial class' Game/FX/*.cs`; `grep 'SpellEffectType\.(AreaOfEffect|Beam)' SpellEffectConfigurations.cs` → none |
| "School grouping: spells are automatically grouped by school" | grouping exists but is a substring guess on the id and `GetSpellEffectsBySchool` has no callers | `SpellEffectRegistry.cs:183-201`; grep → none |
| "Resource caching: scenes and audio are cached" | only `PackedScene` cached (`_sceneCache`); no audio handling in the factory | `SpellEffectFactory.cs:13, 150-160` |
| "Network synchronization for multiplayer" listed as future | positions/timings already come from server snapshots (`pending_impacts`, `due_tick`, `end_tick`) | `SpellEffectManager.DuelV2.cs:77-106` |
| Example `Stop()` for static effect frees the node | consistent with code, kept | — |

### Game/README.md → see §5.

### Cross-note

- `staging/client/architecture.md:73` says "The full pattern (interfaces, examples, logging) is in `client:Application/CQRS/README.md`; it is accurate." It is not (see above) and the README is scheduled for deletion. Replace that sentence with a link to `[[cqrs]]`. Same file's frontmatter `sources` lists the README.

## 3. Code smells / dead code noticed

- `SpellEffectFactory.SetupAutoCleanup` creates a `Timer` per effect that never fires anything (`SpellEffectFactory.cs:140` commented out). Either wire `effect.Stop()` or delete the method and `AutoCleanup`.
- `SpellEffectConfiguration.RenderPriority` is never read.
- `SpellEffectRegistry` school grouping (`_schoolRegistry`, `GetSpellSchool`, `GetSpellEffectsBySchool`) and `UnregisterSpellEffect`, `ClearRegistry`, `GetRegistryCount` have no callers.
- `SpellEffectManager.cs:75-197`: legacy opcode-11/13/15 handlers (`OnSpellCasting`, `OnSpellAccepted`, `OnGameEffectRemoved`) and the `_userEffects` tracking are unreachable since combat v1 opcodes are dropped in `MatchMessageHandler`. Their `using`s still pull in `GameplaySpellAccepted`, `GameEffectRemoved`, `GameplaySpellCasting` messages.
- `SpellEffectManager` registers into a **static** registry every time a controller is created; the registry is never cleared, and the second construction silently overwrites. `VfxTestScreen` does the same guarded by `IsSpellEffectRegistered`.
- `SpellAnimationController.TestHealSpell` creates a `new SpellEffectFactory()` without `SetSceneTree`, so the instantiated node is never added to the tree (`SpellAnimationController.cs:98`).
- `SpellAnimationController.cs:3` has `using hexbane.Application.ArcaneDuel;` (a parent namespace with no types of its own); harmless but misleading.
- `Game/FX/Reflection.cs` + `Reflection.tscn`, `AquaPulse`, `EmberBurst`, `FrostCut`, `Spark`, `Stoneguard` scenes exist but are either unregistered or unmapped for the 14-spell catalog (already noted in `spell-vfx-configuration.md`).
- CQRS: `ICqrsDispatcher` exposes sync `Dispatch<TCommand>` but not the other two sync methods that exist on the class; the sync methods block with `GetAwaiter().GetResult()` on socket callbacks (`MatchManager.cs:128`), a deadlock risk if a handler ever awaits back onto the Godot synchronization context.
- CQRS: `CqrsRegistration.RegisterHandlersOfType` logs with raw `GD.Print("[CQRS DEBUG] …")` for every handler at startup (about 31 handlers) regardless of `debugMode`.
- CQRS event infrastructure (`IEvent`, `BaseEvent`, `IEventHandler<>`, `PublishAsync`) is unused; `GameEvents` autoload does that job.
- `Application/CQRS/CqrsRegistration.cs.uid` is a Godot uid sidecar for a non-script file.

## 4. Facts from `Game/README.md` missing in `architecture.md`

Verified against code; quotable as-is.

- "Autoload scene files `Game/Autoloads/GameContext.tscn`, `GameEvents.tscn`, `SceneManager.tscn`, `MenuPlayer.tscn` and `Game/DI/DIHost.tscn` exist, but `project.godot` registers the autoloads by **script** (`*res://Game/DI/DIHost.cs`, `*res://Game/Autoloads/GameContext.cs`, …), not by scene (`project.godot:29-43`). The `.tscn` files are leftovers of the GDScript→C# migration."
- "The C# autoloads were migrated from GDScript. Migration conventions that still hold: signals are declared with `[Signal]` and a delegate named `<Name>EventHandler`, emitted with `EmitSignal(GameEvents.SignalName.<Name>, …)` (`Game/Autoloads/GameEvents.cs:155`, `Game/ScenesV3/Auth/LoginPanel.cs:351`); private fields are `_camelCase`; methods are PascalCase."
- "Cross-autoload access from a node: `GetNode<GameEvents>("/root/GameEvents").UserAuthenticated += OnUserAuthenticated;` or the static `GameEvents.Instance`; scene changes through `SceneManager.ChangeScene(string sceneKey, TransitionType transitionType = TransitionType.Fade)` returning `bool` (`Game/Autoloads/SceneManager.cs:115`)."
- "`GameEvents.UserAuthenticated` carries a single `string userId` (`GameEvents.cs:155`)."
- "`DIHost` emits `ServicesInitialized` after `ServiceBootstrapper.Build()` and disposes the `ServiceProvider` in `_ExitTree` (`Game/DI/DIHost.cs:12-39`); all services are built once, at startup. There is no second 'session services' phase."
- "`SessionContext` (`Core/Auth/SessionContext.cs`, singleton in DI) holds the Nakama `ISession` and `ISocket`: `GetSession/SetSession`, `GetSocket/SetSocket`, `IsAuthenticated()`, `GetUserId()`, `ClearSession()`. Services that need the logged-in user take it by constructor (`CharacterService.cs:28`, `CharacterPlaystyleService.cs:13`)."

## 5. Stale claims in `Game/README.md` (with evidence)

| README claim | Reality | Evidence |
|---|---|---|
| Autoload order `GameContext, GameEvents, SceneManager, DIHost` (lines 6-11, 83-86) | order is `DIHost, DpiScaler, EnvLoader, GameContext, GameEvents, MatchContext, SceneManager, MainThreadInvoker, GodotGTweensContextNode, NotificationManager, MenuPlayer, DevAutoLogin, MCP*` and they are `.cs` scripts, not `.tscn` | `project.godot:29-43` |
| `IAuthenticationService` / `AuthenticationService` (lines 21, 27, 64) | no such types; auth is `ILoginService`/`LoginService`, `IRegisterService`, `ISocialSignIn`, `ISocialAuthGateway` | `Game/DI/ServiceBootstrapper.cs:74-98`; grep → none |
| `GameContext.Instance.AuthService` and `authService.AuthenticateEmail(...)` (lines 36-39) | `GameContext` exposes no services (only `Instance`, `Character`, session helpers); email auth is `LoginService` calling Nakama `AuthenticateEmailAsync` | `Game/Autoloads/GameContext.cs:15,24`; `Application/Authentication/LoginService.cs:68` |
| "Core services" vs "Session services" split, `DIHost.Instance.RegisterSessionServices()` (lines 25-28, 71-74) | no such method; every service is a singleton registered once in `ServiceBootstrapper.ConfigureServices` | `Game/DI/DIHost.cs` (no such member); `ServiceBootstrapper.cs:61-132` |
| "DIHost registers base services in GameContext", "GameContext stores all services" (lines 55, 75) | `DIHost` only builds `ServiceProvider` into static `DIHost.Services`; `GameContext` holds user/character state | `Game/DI/DIHost.cs:33-39` |
| "Startup scene switches to the login screen" (line 56) | `Game/Scenes/` (Startup) no longer exists; main scene is `Game/ScenesV3/Auth/AuthScreen.tscn`; `SceneManager` still routes `startup` to a dead path | `project.godot:18`; `ls Game/Scenes` → missing; `SceneManager.cs:273` |
| `PlaystyleService` (line 28) | `ICharacterPlaystyleService` / `CharacterPlaystyleService` | `ServiceBootstrapper.cs:105` |
| `EmitSignal(GameEvents.SignalName.UserAuthenticated, userId, username)` (line 117) | signal has one parameter `userId` | `GameEvents.cs:155`, `LoginPanel.cs:351` |
| Title "Migracja z GDScript do C#" / instructions to add autoloads by hand in the editor (lines 79-86) | migration finished; autoloads already in `project.godot` | `project.godot:29-43` |

## 6. Open questions

- Should the inert `AutoCleanup` timer in `SpellEffectFactory` be wired back (`effect.Stop()` on timeout) or removed together with the flag? The FX scenes currently manage their own lifetime; enabling it would double-free scenes that already `QueueFree()` in `Stop()`.
- Are the ten commented-out `Create*()` configurations (`SpellEffectConfigurations.cs:20-37`) intentionally disabled for the duel v2 catalog (fireball, explosion, paralyze, arcane_shield, venom_shot are still mapped in `VisualId`), or leftovers? With them disabled, `firebolt`, `delayed_hex`, `paralysis`, `barrier`, `consume_venom` and `greater_heal` produce no impact visual (`ShowVisual` returns null on a missing template).
- Is the unused CQRS event bus (`IEvent`/`PublishAsync`) meant to stay as a future facility, or should it be removed in favour of `GameEvents`?
- The leftover autoload `.tscn` files in `Game/Autoloads/` and `Game/DI/DIHost.tscn`: keep or delete?
