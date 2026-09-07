---
type: project
project: Hexbane
domain: projects
status: archived
created: 2026-08-31
updated: 2026-09-07
archived: 2026-09-07
original-path: 10_Projects/hexbane/architektura-klienta.md
superseded-by: "[[_index]] (10_Projects/hexbane/docs/)"
tags: [hexbane, godot, csharp, architecture]
aliases: [hexbane-client]
---

> [!warning] Zarchiwizowane 2026-09-07 — opis sprzed przeprojektowania duel_v2. Aktualna dokumentacja: `10_Projects/hexbane/docs/` (patrz `[[_index]]`). Audyt rozbieżności: `[[2026-09-07-vault-notes-audit]]`.

# Hexbane — architektura klienta (Godot 4.5 / C#)

Stan na 2026-08-31 (`elanon1/hexbane@01ec454`). Część projektu [[10_Projects/hexbane/_state|Hexbane]]. Szczegóły z numerami linii: `thoughts/shared/research/2026-08-31-hexbane-client-server-recon.md` w repo.

## Warstwy i start

- `Game/` (sceny, autoloady, UI) → `Application/` (CQRS, serwisy, handlery Nakama) → `Core/` (modele). W praktyce `Application` i nawet `Core` (`Core/Match/MatchState.cs`) sięgają w górę do `Game.Autoloads`/`DIHost` (service locator).
- Główna scena: `Game/ScenesV3/Auth/AuthScreen.tscn`. Viewport 2400×1080, `canvas_items`/`expand`, `gl_compatibility`, Android.
- `hexbane.csproj`: `Godot.NET.Sdk/4.7.0`, `net9.0`, `NakamaClient 3.16.0`, DI/Logging 9.0.7, Newtonsoft. **`.env` pakowany do builda jako `Content`.**
- Autoloady (kolejność!): `DIHost` → `DpiScaler` → `EnvLoader` → `GameContext` → `GameEvents` → `MatchContext` → `SceneManager` → `MainThreadInvoker` → `GodotGTweensContextNode` → `NotificationManager` → `MenuPlayer` → `DevAutoLogin` → addony MCP.

## DI + CQRS

- `Game/DI/ServiceBootstrapper.cs`: singletony `SessionContext`, `INakamaClientManager`, `ILoginService`, `IRegisterService`, `ICharacterService`, `ISpellService`, `IRaceService`, `ICharacterPlaystyleService`; **keyed `IMatchManager`**: `ad_normal` (PvP), `ad_ai` (bot), `create_character` (Endless Story); moduł notyfikacji (7 listenerów), moduł social; `AddCqrs` skanuje całe assembly; `IMatchDataHandler<T>` rejestrowane osobno.
- `Application/CQRS/`: `ICommand`, `IQuery<T>`, `IEvent`; `CqrsDispatcher` rozwiązuje handlery przy dispatchu (transient).
- Komendy: `CreateCharacter`, `AllocateStatPoints`, `LearnSpell`, `Add/Remove/BlockFriend`, `DeleteNotifications` + wychodzące do meczu (`CastSpell`, `Meditate`, `Quit`, `ClientReady`, `LobbySpellSelection`, `ChoiceSelected`). Zapytania: postać (`get_my_character`, `get_character_by_id`, `get_progression`), rasy (`get_races`, cache w `RaceService`), zaklęcia (`get_player_spells`, `get_spell`, `get_spellbook`, `get_starter_spells`), social, notyfikacje, `start_story`.

## Kontekst i zdarzenia

- `GameContext`: `Character`, `IsAuthenticated`, klucz serwera; timer 30 s sprawdza sesję/socket i wylogowuje.
- `GameEvents`: mieszanka `Action` (wysokoczęstotliwościowe: `OnPlayerUpdate`, `OnPlayerEffectUpdate`, `OnGameplaySpellCasting`) i sygnałów Godota (`MatchStatusChanged`, `GameOver`, `SpellRelease`, `GameLog`, połączenie).
- `MatchContext`: `MatchId`, `MatchState` (`Idle/Searching/MatchFound/Connecting/Lobby/LobbySelected/InMatch/MatchEnded/Error`), `Me/Enemy`, `MeSnapshot`, `SpellSlotOrder`, `IsAiMatch`, bufor `PendingGameLoad*`.
- `SceneManager`: mapa kluczy → `.tscn`; auto-nawigacja po `MatchStatusChanged` (`Lobby`→lobby, `InMatch`→`GameHud/Main.tscn`, `MatchEnded`→game_over). **Martwe klucze**: `startup`, `autoload`, `debug_rpcs` (folder `Game/Scenes/` nie istnieje).
- `MainThreadInvoker`: kolejka z wątku socketu na główny.
- `EnvLoader` (`.env`): `NAKAMA_SERVER` (`local|synology|prod`), `NAKAMA_HOST`, `DEV_AUTO_LOGIN`, `DEV_AUTO_LOGIN_MODE` (`existing|new_character`), `AUTH_EMAIL/PASSWORD`, `DEBUG_SCENE`. Brak `.env.example`.

## Nakama

- `Application/Nakama/NakamaClientManager.cs`: serwery zaszyte w kodzie — `local` (`NAKAMA_HOST:7350`), `synology` (`192.168.1.9:7350`), `prod` (`hexbane.elanon.pl:443` https). Socket `Closed` → reset + logout; **brak auto-reconnect**.
- Auth: email/hasło (`AuthenticateEmailAsync`), rejestracja = ta sama metoda z `create=true`; fallback device-id.
- Przepływ wiadomości meczu: socket → `MatchManager.OnMatchData` → `MainThreadInvoker` → `MatchMessageCommand` → `MatchMessageHandler` (mapa opcode → DTO z `[MatchOpcode]` → `IMatchDataHandler<T>`) → `GameEvents` → ekrany. Pełna tabela: [[10_Projects/hexbane/protokol-klient-serwer]].
- Matchmaking PvP: `AddMatchmakerAsync("", 2, 2)`; bot: RPC `create_ai_arcane_duel`.

## Ekrany (`Game/ScenesV3/`, UI budowane w C#)

Auth (Login/Register + selektor serwera) · Dashboard (karta postaci, siatka zaklęć, news; `ModeOverlay` PvP/AI z 10 s accept/decline) · CreateCharacter (5 kroków: imię → staty 400 pkt → rasa z podglądem `RaceAnimationPreview` → 4 zaklęcia startowe → podsumowanie) · CharacterDetail (staty/skille/spellbook, nauka zaklęć za MP) · Lobby (draft naprzemienny + **tryb układania** zaklęć drag&drop → `SpellSlotOrder`) · GameHud (`Main.tscn`: parallax, kamera z shake, 2× `Player.tscn`, HUD: 10 slotów L/P z `CastSweepOverlay`, timer, log, paski HP/many, `EffectSlot` z odliczaniem z serwera, winieta; **medytacja = swipe w górę**) · GameOver · Settings (`user://settings.cfg`) · Social · News · VfxTest.

**Osierocone**: `GameHud/ArcaneDuel/Main.tscn` + `SpellBar/PlayerBars/Meditate/GameTimer/...`; dev: `Dev/VitraelPreview.gd`, `RaceTest/*`. **GTweens** (autoload) ma zero użyć — wszędzie natywny `Tween`.

## VFX zaklęć

- `Core/Spells/ISpellEffect.cs` (`Projectile/Static/AreaOfEffect/Beam`), `SpellEffectRegistry`, `Application/Modules/Spell/Effects/{SpellEffectConfigurations,SpellEffectFactory,SpellEffectManager}.cs`, sceny `Game/FX/*.tscn` (20).
- Aktywne konfiguracje (9): `magic_sparkle`, `heal`, `restore`, `cure`, `flamestrike`, `gust`, `stoneguard`, `mirror_ward`, `spark`; 11 zakomentowanych (`fireball`, `poison_dart`, `reflection`, `arcane_shield`, `paralyze`, …). `SpellEffectManager` reaguje na `OnGameplaySpellAccepted` (efekt caster → `FinalTarget`) i `OnGameEffectRemoved` (`consumed`/`expired` → shatter/fade tarcz).
- `FlashCaster` z CLAUDE.md nie istnieje.

## Design system

`docs/DESIGN_SYSTEM.md`: tła brązowo-czarne (R>G>B, alpha 0.45–0.95), akcent `#D9591F`, Overpass (+ Cinzel Decorative dla logo), skala rozmiarów 14–64, promienie 8/10/12, kolory statów STR/INT/DEX = czerwono-pomarańczowy/niebieski/zielony. Agent `ui-designer` w `.claude/agents` pilnuje spójności.
