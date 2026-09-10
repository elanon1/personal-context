---
type: project
project: Hexbane
area: client
domain: [projects]
status: active
created: 2026-09-07
updated: 2026-09-10
verified: 2026-09-10
tags: [hexbane, client, architecture, godot]
sources: ["client:CLAUDE.md", "client:AGENTS.md", "client:project.godot", "client:Game/Autoloads/SceneManager.cs", "client:Game/DI/ServiceBootstrapper.cs"]
---

# Client architecture (Godot 4.7 + C# .NET 9)

Repo: `/Users/elanon/RiderProjects/hexbane`. Godot.NET.Sdk **4.7.0** (`hexbane.csproj:1`), project features `4.7, C#, Mobile` (`project.godot:20`). Backend: Nakama via `NakamaClient 3.16.0`, DI via `Microsoft.Extensions.DependencyInjection 9.0.7`, `Newtonsoft.Json 13.0.3` (`hexbane.csproj:46-50`). The unused GTweens/Godot tween plugin and autoload were removed on 2026-09-08; animations use Godot Tween.

## Layers

```
Game/         Godot integration: autoloads, DI host, ScenesV3 screens, FX scenes
Application/  CQRS handlers, feature services, Nakama match managers, match message handlers
Core/         domain: Auth, Characters, Common (Opcodes), Match (DuelV2, PlayerSnapshot), Spells, Tutorial
```
Dependencies flow `Game → Application → Core`. Tests live in `Tests/` and are excluded from the main assembly (`hexbane.csproj:15`).

## Autoloads (order from `project.godot:29-43`)

| # | Autoload | Script | Purpose |
|---|---|---|---|
| 1 | DIHost | `Game/DI/DIHost.cs` | builds the ServiceProvider (`ServiceBootstrapper`) |
| 2 | DpiScaler | `Game/Autoloads/DpiScaler.cs` | phone scaling; ScenesV3 screens use `Game/ScenesV3/_Common/ResponsiveLayout.cs` compact mode (phone viewport ≈ 1360×612) |
| 3 | EnvLoader | `Game/Autoloads/EnvLoader.cs` | loads `res://.env` unless `HEXBANE_IGNORE_ENV_FILE=1` (`EnvLoader.cs:15`) |
| 4 | GameContext | `Game/Autoloads/GameContext.cs` | authenticated user + current character; periodic `CheckSessionHealth` |
| 5 | GameEvents | `Game/Autoloads/GameEvents.cs` | event bus (25 `public Action` fields incl. `OnDuelSnapshot`, `OnDuelEvent`, `OnPlayerUpdate`) |
| 6 | MatchContext | `Game/Autoloads/MatchContext.cs` | `MatchId`, `IsAiMatch`, `Combat` (`DuelState`), `Me`/`Enemy`, `SpellSlotOrder` |
| 7 | SceneManager | `Game/Autoloads/SceneManager.cs` | route table, history, Android Back handling |
| 8 | MainThreadInvoker | `Game/Autoloads/MainThreadInvoker.cs` | marshals socket callbacks to the main thread |
| 9 | NotificationManager | `Game/Autoloads/NotificationManager.cs` | Nakama notifications |
| 10 | MenuPlayer | `Game/Autoloads/MenuPlayer.cs` | menu/battle music |
| 11 | DevAutoLogin | `Game/Autoloads/DevAutoLogin.cs` | `DEV_AUTO_LOGIN=true` + `DEV_AUTO_LOGIN_MODE=existing|new_character` (`DevAutoLogin.cs:21-29`) |
| 12–14 | MCPScreenshot, MCPInputService, MCPGameInspector | `addons/godot_mcp/*.gd` | editor MCP tooling only |

CLAUDE.md lists only six autoloads; the table above is the real set.

## Dependency injection (`Game/DI/ServiceBootstrapper.cs`)

Singletons: `SessionContext`, `INakamaClientManager`, `IAuthValidator`, `ISocialSignIn` (factory `CreateSocialSignIn`, lines 47-59: `GoogleOAuthSignIn` when `AuthConfig.HasGoogleClient`, else `NullSocialSignIn`), `ISocialAuthGateway → GoogleAuthGateway`, `ILoginService`, `IRegisterService`, `TutorialSession`, `ICharacterService`, `ISpellService`, `IRaceService`, `ICharacterPlaystyleService` (lines 71-105). Notifications and Social modules register through their own extension methods (lines 114-118).

Keyed `IMatchManager` (lines 107-110):

| Key | Implementation |
|---|---|
| `ad_normal` | `Application/ArcaneDuel/Normal/MatchManager` (PvP matchmaker) |
| `ad_ai` | `Application/ArcaneDuel/Bot/BotMatchManager` |
| unkeyed | `MatchManager` |

Access: `DIHost.Services.GetService<T>()` / `GetRequiredKeyedService<IMatchManager>("ad_ai")`.

## CQRS (`Application/CQRS/`)

`AddCqrs(enableLogging, debugMode, assemblies)` (`CqrsRegistration.cs:25`) reflects over the given assemblies and registers every `ICommandHandler<T>`, `ICommandHandler<T,R>`, `IQueryHandler<Q,R>`, `IEventHandler<E>` as transient. Bootstrapper passes the Application assembly (`ServiceBootstrapper.cs:120-128`). `IMatchDataHandler<T>` implementations are registered separately by `RegisterMatchDataHandlers` (lines 134-149).

Dispatcher API (`Application/CQRS/Bus/CqrsDispatcher.cs`):
```csharp
await dispatcher.DispatchAsync(new CastSpellCommand(...));            // ICommand
var r = await dispatcher.DispatchAsync<Cmd, Result>(cmd);            // ICommand<Result>
var q = await dispatcher.QueryAsync<GetSpellsQuery, List<Spell>>(q); // IQuery<Result>
await dispatcher.PublishAsync(new PlayerCreatedEvent(...));          // IEvent, many handlers
```
The full pattern (interfaces, examples, logging) is in [[cqrs]] (the old `Application/CQRS/README.md` was removed on 2026-09-07; its examples did not compile).

## Nakama message flow

```
ISocket.ReceivedMatchState
  → IMatchManager.OnMatchData (ad_normal / ad_ai)
  → dispatcher.DispatchAsync(new MatchMessageCommand(opcode, data))
  → MatchMessageHandler.HandleAsync   (Application/Match/Incoming/MatchMessageHandler.cs)
       opcode 30..32  → DuelProtocol.Receive(...)   (line 22, combat v2, no handler class)
       opcode 11..15, 21..28 → dropped (retired combat v1, line 23 and 102)
       other          → IMatchDataHandler<T> found by [MatchOpcode] attribute
  → handler invokes GameEvents → UI reacts (marshalled via MainThreadInvoker)
```
Opcode enum: `Core/Common/Enums/Opcodes.cs` (still declares the retired 11–15/21–28 names). Protocol details: [[opcodes]], [[combat-v2]], client side [[duel-v2-client]].

Outgoing commands: `Application/Match/Outgoing/{CastSpell,ClientReady,Meditate,Quit,SpellSelection}`. `ClientReadyCommand.ToPayload()` sends `{combat_protocol:2, user_id, event_name}` (`ClientReadyCommand.cs:17-21`).

## Scene routes (`Game/Autoloads/SceneManager.cs:14-36`)

| Key | Scene | Note |
|---|---|---|
| `login` | `Game/ScenesV3/Auth/AuthScreen.tscn` | main scene (`project.godot:18`) |
| `dashboard`, `main_menu` | `Game/ScenesV3/Dashboard/DashboardScreen.tscn` | |
| `character_creation` | `Game/ScenesV3/CreateCharacter/CreateCharacterScreen.tscn` | 5-step wizard |
| `character_detail` | `Game/ScenesV3/CharacterDetail/CharacterDetailScreen.tscn` | Summary / Stats / Spellbook |
| `news`, `settings`, `social` | `Game/ScenesV3/{News,Settings,Social}/*Screen.tscn` | |
| `lobby` | `Game/ScenesV3/Lobby/LobbyScreen.tscn` | spell draft |
| `normal_game` | `Game/ScenesV3/ReferenceDuel/MainReference.tscn` | PvP and AI duels |
| `game_over` | `Game/ScenesV3/GameOver/GameOverScreen.tscn` | |
| `vfx_test` | `Game/ScenesV3/VfxTest/VfxTestScreen.tscn` | no caller in code |
| `startup`, `autoload`, `debug_rpcs` | `Game/Scenes/...` | **dead**: `Game/Scenes/` no longer exists |

Not in the table but routed by file: `Game/ScenesV3/Tutorial/TutorialScreen.tscn` (`GoToCharacterCreation`, line 297-300). `GoToDashboard` (lines 281-287) sends players without a completed training to the tutorial first. Match status → scene: `Lobby → lobby`, `InMatch → normal_game`, `MatchEnded → game_over`, `Idle/Error → main menu` (lines 541-580).

Android Back: `application/config/quit_on_go_back=false` (`project.godot:19`); `SceneManager._Notification` (line 365) ignores Back on `Startup, AutoloadScreen, AuthScreen, DashboardScreen, TutorialScreen, LobbyScreen, MainReference, GameHudScreen, GameOverScreen` and otherwise navigates back or to the dashboard.

Other scene trees: `Game/ScenesV3/GameHud/` (pre-reference HUD, used only by `Dev/GameHudPreview` and `Dev/HudPreview`), `Game/ScenesV3/Dev/` (dev/verification scenes, see [[vfx-and-race-animation]]), `Game/FX/` (spell effect scenes).

## Configuration resolution

| Value | Order |
|---|---|
| Nakama host/port | `.env` `NAKAMA_HOST`/`NAKAMA_PORT` → `hexbane/network/local_host` project setting → `127.0.0.1:7350`, server key `defaultkey` (`NakamaClientManager.cs:109-115, 154-155`) |
| Google OAuth | env `GOOGLE_CLIENT_ID/SECRET/LOOPBACK_PORT` → `hexbane/auth/google_*` settings (`AuthConfig.cs:29-51, 91-97`) |
| Race sprite set | `hexbane/graphics/race_sprites` = `auto|hd|sd`; auto = SD on `mobile`/`web` features (`RaceAnimationPreview.cs:34, 56-64`) |
| Dev login | `.env` `DEV_AUTO_LOGIN`, `DEV_AUTO_LOGIN_MODE`, `AUTH_EMAIL`, `AUTH_PASSWORD` |

`.env` keys present locally: `DEBUG_SCENE, NAKAMA_SERVER, NAKAMA_HOST, DEV_AUTO_LOGIN, DEV_AUTO_LOGIN_MODE, AUTH_EMAIL, AUTH_PASSWORD, GOOGLE_CLIENT_ID, GOOGLE_CLIENT_SECRET`. `.env` is never packed into exports (see [[deploy-android]]).

Display: 2400×1080 viewport, `canvas_items` stretch, `expand` aspect, per-pixel transparency **off** (`project.godot:47-55`, must stay off, see [[deploy-android]]).

## Test projects

| Project | Run | Covers |
|---|---|---|
| `Tests/DuelV2/DuelV2.csproj` | `dotnet run --project Tests/DuelV2/DuelV2.csproj [ai|pvp]` | offline duel_v2 model checks; with an argument a live SDK smoke that requires `NAKAMA_URL` and the seeded `elf@test.pl` / `dark_elf@test.pl` accounts (`Live.cs:72-77`) |
| `Tests/Tutorial/Tutorial.csproj` | `dotnet run --project Tests/Tutorial/Tutorial.csproj` | `TrainingBattle` steps and `ProgressionLesson.ShouldStart` |
| `Tests/Tutorial/live_rpc.py` | `python3 Tests/Tutorial/live_rpc.py` | `tutorial` RPC against `127.0.0.1:7350` with a disposable device account |
| Godot headless scenes | `Game/ScenesV3/Dev/*.tscn`, `ReferenceDuel/ReferenceVerification.tscn` | see [[duel-v2-client]], [[client-tutorial]], [[vfx-and-race-animation]] |

Build: `dotnet build hexbane.csproj`. No formatter/linter; `.editorconfig` only sets UTF-8.

## Conventions

Four-space indent, file-scoped namespaces, `_camelCase` private fields, PascalCase public members, scene sub-components may be `_Name.cs`, signal delegates end in `EventHandler`. Services via `DIHost.Services` or `GameContext.Instance`.

## Source of truth in code
- `client:project.godot` — autoload order, `[hexbane]` settings, display/stretch, main scene
- `client:Game/DI/ServiceBootstrapper.cs` — every DI registration, keyed match managers, social sign-in factory
- `client:Application/CQRS/CqrsRegistration.cs`, `Bus/CqrsDispatcher.cs` — handler discovery and dispatch
- `client:Application/Match/Incoming/MatchMessageHandler.cs` — opcode routing and retired-opcode filter
- `client:Game/Autoloads/SceneManager.cs` — route table, tutorial gating, Back handling
- `client:Application/Nakama/NakamaClientManager.cs`, `Application/Authentication/AuthConfig.cs`, `Game/Autoloads/EnvLoader.cs` — configuration resolution

## Autoload and signal conventions (from the GDScript → C# migration)

- `project.godot` registers autoloads by **script** (`*res://Game/DI/DIHost.cs` …); the leftover
  `Game/Autoloads/*.tscn` and `Game/DI/DIHost.tscn` scene files are not what runs.
- Signals: `[Signal] public delegate void <Name>EventHandler(...)`, emitted with
  `EmitSignal(GameEvents.SignalName.<Name>, …)`. `GameEvents.UserAuthenticated` carries a single
  `string userId` (`client:Game/Autoloads/GameEvents.cs:155`).
- Cross-autoload access: `GetNode<GameEvents>("/root/GameEvents")` or the static `GameEvents.Instance`;
  scene changes only through `SceneManager.ChangeScene(sceneKey, transition)`.
- `DIHost` builds the `ServiceProvider` once at startup (`ServiceBootstrapper.Build()`), emits
  `ServicesInitialized`, and disposes it in `_ExitTree` (`client:Game/DI/DIHost.cs:12-39`). There is no
  second "session services" phase; every service is a singleton.
- `SessionContext` (`client:Core/Auth/SessionContext.cs`, DI singleton) owns the Nakama `ISession` /
  `ISocket` (`GetSession/SetSession`, `GetSocket/SetSocket`, `IsAuthenticated()`, `GetUserId()`,
  `ClearSession()`); `GameContext` holds only user/character state, no services.
- Auth services are `ILoginService`, `IRegisterService`, `ISocialSignIn`, `ISocialAuthGateway`
  (`client:Game/DI/ServiceBootstrapper.cs:74-98`); playstyles are `ICharacterPlaystyleService`.

## 2026-09-08 cleanup

Client storytelling was removed: `Application/EndlessStory`, `Application/Modules/Story`, incoming/outgoing `EndlessStory`, DI key `create_character`, `NarrationUpdated` and client opcode constants 100/101. The current character creation/tutorial flow remains. No server repository was changed in this client task.

Retired combat handlers (11–15, 21–28) and unused DTOs were removed. Casting/accepted/failed DTOs remain because current development previews and HUD compatibility code still consume them; their old network handlers are gone. The dispatcher still drops the retired numeric ranges.

Spell VFX consists of MirrorWard (formation/shatter audio restored at the user’s request) and procedural Magic Arrow / Firebolt projectile impacts with one shared lifecycle; new cast/gesture/meditation effects and current catalog icons remain. See [[spell-effect-system]].

## Duel launch readiness (2026-09-10)

See [[duel-v2-client#Arena launch/countdown repair (2026-09-10)]]. The active HUD acknowledges its first rendered frame and completed scene transition; the server waits for every human arena before counting down. MatchContext also buffers early opcode 16 values alongside game-load data.
