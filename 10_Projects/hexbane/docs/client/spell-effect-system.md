---
type: project
project: Hexbane
area: client
domain: [projects]
status: active
created: 2026-09-07
updated: 2026-09-07
verified: 2026-09-07
tags: [hexbane, client, spells, vfx, architecture]
sources: ["client:Application/ArcaneDuel/README.md"]
---

# Spell effect system (architecture and API)

How a spell id becomes a VFX node on screen. Which spells have which assets, the id mapping table and the cast-gesture presets are in [[spell-vfx-configuration]]; this note covers only the types and the runtime path.

## Layers

| Layer | File | Role |
|---|---|---|
| Core | `Core/Spells/ISpellEffect.cs` | `ISpellEffect` + four typed interfaces |
| Core | `Core/Spells/SpellEffectRegistry.cs` | `SpellEffectType`, `SpellEffectDirection`, `SpellEffectConfiguration`, static `SpellEffectRegistry` |
| Application | `Application/Modules/Spell/Effects/SpellEffectConfigurations.cs` | the built-in configurations (`GetDefaultConfigurations()`) |
| Application | `Application/Modules/Spell/Effects/SpellEffectFactory.cs` | loads/caches the scene, instantiates, positions, plays |
| Application | `Application/Modules/Spell/Effects/SpellEffectManager.cs` + `SpellEffectManager.DuelV2.cs` | registers configs, listens to duel events, tracks live effects |
| Game | `Game/ScenesV3/GameHud/ArcaneDuel/Components/SpellAnimationController.cs` | Godot node owning the manager, resolves player positions |
| Game | `Game/FX/*.tscn` + `*.cs` | the effect scenes |

`Application/ArcaneDuel/` itself contains only the two match managers (`Normal/MatchManager.cs`, `Bot/BotMatchManager.cs`); the effect code moved to `Application/Modules/Spell/Effects/`.

## Effect interfaces (`Core/Spells/ISpellEffect.cs`)

```csharp
public interface ISpellEffect { bool IsPlaying { get; } void Stop(); Vector2 GetPosition(); }

public interface IProjectileEffect : ISpellEffect {
    void Launch(Vector2 from, Vector2 to, float duration, Dictionary<string, object> parameters = null);
    event Action<Vector2> OnTargetReached; event Action OnDestroyed; }

public interface IStaticEffect : ISpellEffect {
    void Play(Vector2 position, float duration, Dictionary<string, object> parameters = null);
    event Action OnStarted; event Action OnFinished; }

public interface IAreaOfEffectEffect : ISpellEffect {
    void Play(Vector2 center, float radius, float duration, Dictionary<string, object> parameters = null);
    event Action<Node2D> OnEntityEntered; event Action<Node2D> OnEntityExited; }

public interface IBeamEffect : ISpellEffect {
    void CreateBeam(Vector2 from, Vector2 to, float duration, Dictionary<string, object> parameters = null);
    event Action<Vector2, Node2D> OnBeamHit; }
```

Every scene in `Game/FX/` is a `Node2D` implementing exactly one of `IProjectileEffect` (AquaPulse, EmberBurst, Fireball, FrostCut, Gust, MagicSparkle, PoisonDart, Spark, VenomShot) or `IStaticEffect` (ArcaneShield, Cure, Explosion, FlameStrike, GreatHeal, Heal, MirrorWard, Paralyze, Reflection, Restore, Stoneguard). **No `IAreaOfEffectEffect` or `IBeamEffect` implementation and no configuration of those types exist**; the enum cases and the factory branches for them are unused.

## Configuration (`Core/Spells/SpellEffectRegistry.cs:53-78`)

```csharp
public class SpellEffectConfiguration {
    public string SpellId;                  // registry key = FX id
    public SpellEffectType EffectType;      // Projectile | Static | AreaOfEffect | Beam
    public SpellEffectDirection Direction;  // CasterToTarget | TargetToCaster | OnCaster | OnTarget | FromCaster | FromTarget
    public string EffectScenePath;          // "res://Game/FX/Heal.tscn"
    public float Duration;                  // seconds
    public Dictionary<string, object> Parameters = new();
    public bool AutoCleanup = true;
    public int RenderPriority = 0;          // never read
}
```

There is no `AudioPath`; sounds are loaded by the FX scene itself (see [[spell-vfx-configuration]]).

`SpellEffectRegistry` is a process-wide static map `SpellId → SpellEffectConfiguration`: `RegisterSpellEffect`, `RegisterSpellEffects`, `GetSpellEffect(id)` (null when unknown), `IsSpellEffectRegistered`, `GetAllSpellEffects`, `UnregisterSpellEffect`, `ClearRegistry`, `GetRegistryCount`. It also groups entries by a "school" guessed from substrings of the id (`fire→Pyromancy`, `ice→Cryomancy`, … default `Universal`, lines 183-201) for `GetSpellEffectsBySchool`; nothing calls that.

Registration happens when a `SpellEffectManager` is constructed (`SpellEffectManager.cs:43-52`) and again in `VfxTestScreen._Ready` (`VfxTestScreen.cs:31-36`), from `SpellEffectConfigurations.GetDefaultConfigurations()`. Half of the `Create*()` entries are commented out of that list (`SpellEffectConfigurations.cs:16-38`), so e.g. `fireball`, `explosion`, `paralyze`, `arcane_shield`, `venom_shot` are **not registered** at runtime even though their factories and scenes exist. `SpellEffectManager.RegisterCustomSpellEffect(config)` (also exposed on `SpellAnimationController`) adds one more at runtime.

## Factory (`SpellEffectFactory.cs`)

`CreateSpellEffect(config, casterPosition, targetPosition)`:

1. `GD.Load<PackedScene>(config.EffectScenePath)`, cached per path (`GetScene`, lines 150-160).
2. `Instantiate<Node2D>()`, added as a child of `SceneTree.Root` (needs `SetSceneTree` first, else error and the node is orphaned), cast to the interface matching `EffectType`; a scene that does not implement it yields `null` (lines 52-87).
3. Start/end points from `Direction` (lines 121-133): `CasterToTarget` = caster→target, `TargetToCaster` reversed, `OnCaster`/`FromCaster` = caster for both, `OnTarget`/`FromTarget` = target for both. Then `Launch` / `Play` / `Play(radius from Parameters["radius"], default 100)` / `CreateBeam` with `config.Duration` and `config.Parameters` (lines 94-112).
4. `AutoCleanup && Duration > 0` adds a one-shot `Timer` child to the node, but its `Timeout` handler is commented out (`SetupAutoCleanup`, lines 135-148). **Auto-cleanup does nothing**; each FX scene frees itself (`Stop()` / own timers), and the manager stops impacts after 0.5 s.

## Runtime path (duel v2)

`SpellAnimationController._Ready` creates `new SpellEffectManager(this)` and disposes it on `_ExitTree`. The manager subscribes to `GameEvents.Instance.OnDuelEvent` and `OnDuelSnapshot` (`SpellEffectManager.cs:54-70`); the payload types are `Core/Match/DuelV2.cs` (`DuelEvent`, `DuelSnapshot`, `DuelImpact`), see [[duel-v2-client]].

`SpellEffectManager.DuelV2.cs`:

- `VisualId(serverId)` maps the server catalog id to the FX id (lines 15-20; table in [[spell-vfx-configuration]]). Unknown ids pass through unchanged.
- `ShowVisual(id, owner, target, duration, travel)` (lines 21-41) copies the registered template into a fresh config with `Direction = travel ? CasterToTarget : OnTarget`, `Duration = max(0.05, duration)`, and creates it via the factory at the target (or caster→target when travelling). A `MirrorWard` is additionally attached to the target's `RaceSpriteAnimator`.
- **Snapshot** (`OnDuelSnapshot`, lines 77-106): for every player effect whose `Key` is `shield | reflection | paralyze | regeneration` and not yet shown, spawn a sustained status visual keyed by `effect_instance_id`, duration = ticks remaining until `end_tick`; stop visuals whose instance id disappeared. For every `pending_impacts` entry not yet shown, spawn a travelling visual keyed by `action_id`, duration until `due_tick`; stop flights no longer pending. `snapshot.over` → `StopAllEffects()`.
- **Events** (`OnDuelEvent`, lines 42-76): `effect_removed` stops the status visual (MirrorWard shatters on reason `consumed | broken | depleted`, otherwise fades); `spell_impact` stops the matching flight and shows a 0.35 s impact visual at the target, removed after 0.5 s (none for `mirror_ward`, which is the sustained status); `spell_reflected` shatters the reflector's ward toward the new target and draws a 0.25 s `Line2D` bounce.

Positions come from `SpellAnimationController.GetPlayerPosition(userId)` / `GetPlayerAnimator(userId)`, which scan the exported `_playersRootPath` for `Player` children by `PlayerId`.

The pre-v2 entry points `OnSpellCasting(GameplaySpellCastingMessage)`, `OnSpellAccepted(GameplaySpellAcceptedMessage)` and `OnGameEffectRemoved(GameEffectRemovedMessage)` remain in `SpellEffectManager.cs:75-197` but nothing calls them (opcodes 11-15 are dropped by `MatchMessageHandler`, see [[client-architecture]]).

## Adding an effect type or scene

1. Scene: `Game/FX/<Name>.tscn` with a `partial class <Name> : Node2D, IProjectileEffect|IStaticEffect`. Implement `Launch`/`Play`, set `IsPlaying`, raise `OnDestroyed`/`OnFinished`, and `QueueFree()` yourself in `Stop()` and at the end of the animation.
2. Config: a `Create<Name>()` in `SpellEffectConfigurations.cs` and an entry in `GetDefaultConfigurations()`.
3. Mapping: if the server id differs from the FX id, add it to `VisualId` (and the icon path, see [[spell-vfx-configuration]]).
4. Preview: `Game/ScenesV3/VfxTest/VfxTestScreen.tscn` lists every registered config.

## Source of truth in code
- `client:Core/Spells/ISpellEffect.cs` — effect contracts
- `client:Core/Spells/SpellEffectRegistry.cs` — enums, configuration class, static registry
- `client:Application/Modules/Spell/Effects/SpellEffectFactory.cs` — instantiation, direction → positions, (inert) auto-cleanup
- `client:Application/Modules/Spell/Effects/SpellEffectManager.cs`, `SpellEffectManager.DuelV2.cs` — registration, snapshot/event driven spawning, id mapping
- `client:Application/Modules/Spell/Effects/SpellEffectConfigurations.cs` — which FX ids are registered
- `client:Game/ScenesV3/GameHud/ArcaneDuel/Components/SpellAnimationController.cs` — scene owner, player lookup
- `client:Game/FX/*.cs` — the implementations
