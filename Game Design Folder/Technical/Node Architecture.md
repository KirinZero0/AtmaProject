# Node Architecture

High-level Godot scene tree. No scripts here — just structure and signal flow.

---

## Autoloads (Singletons)

```
AtmaManager        ← corruption level, mask state (NEUTRAL / CONSUMED / DOMINATED), ending path
SceneManager       ← scene transitions, loading, owns PlayerData (save/load)
```

---

## PlayerData (Resource — no autoload)

Plain `Resource` owned by `SceneManager`. Saved to disk at save points, loaded on game start. Access via `SceneManager.player_data`.

To add a new unlockable: add a key to the relevant dictionary. No other changes needed.

```gdscript
# PlayerData.gd
class_name PlayerData extends Resource

@export var weapons: Dictionary = {
    "bow":      false,
    "dagger":   false,
    "sword":    false,
    "gauntlet": false,
}

@export var traversal: Dictionary = {
    "mystic_sash":    false,  # from Dedari — unlocks Upper Road / Cliff areas
    "spectral_wings": false,  # from Garuda — unlocks cliff summit
}

@export var mantras: Dictionary = {
    "audience_mantra":   false,  # gates Communion Island (Dominated path)
    "rebel_base_mantra": false,  # gates rebel NPC access
}

@export var key_items: Dictionary = {
    "village_key": false,
    # add new key items here
}

func has(category: String, key: String) -> bool:
    return get(category).get(key, false)

func unlock(category: String, key: String) -> void:
    get(category)[key] = true
```

### SceneManager save/load pattern

```gdscript
const SAVE_PATH = "user://save.tres"

var player_data: PlayerData

func load_game() -> void:
    if ResourceLoader.exists(SAVE_PATH):
        player_data = ResourceLoader.load(SAVE_PATH)
    else:
        player_data = PlayerData.new()

func save_game() -> void:
    ResourceSaver.save(player_data, SAVE_PATH)
```

### Usage at call sites

```gdscript
# check before allowing traversal
if not SceneManager.player_data.has("traversal", "mystic_sash"):
    # block path

# grant on boss kill
SceneManager.player_data.unlock("traversal", "mystic_sash")
SceneManager.save_game()

# check weapon availability in WeaponManager
if SceneManager.player_data.has("weapons", "bow"):
    # add bow to weapon slots
```

---

## Scene Tree

```
Main (Node)
├── World (Node)
│   ├── Level (instanced scene — swapped per location)
│   │   ├── Environment
│   │   └── EnemySpawner
│   └── Player (instanced scene)
└── UI (CanvasLayer)
    ├── HUD
    │   ├── HealthBar
    │   ├── AtmaMeter          ← corruption level visual
    │   ├── MaskStateIndicator ← Neutral / Consumed / Dominated
    │   └── WeaponIndicator    ← active weapon slot
    └── PauseMenu
```

---

## Player Scene

Top-level StateMachine handles character-wide states only. Each weapon is a script that `extends WeaponBase` — attack/dodge/parry are virtual methods overridden per weapon. `WeaponManager` calls `active_weapon.attack()` without knowing which weapon is active (polymorphism).

```
Player (CharacterBody)
├── CollisionShape
├── Sprite / AnimationPlayer
├── StateMachine (Node)             ← character-level states only
│   ├── IdleState
│   ├── MoveState
│   ├── WeaponActionState           ← calls active_weapon.attack() / .dodge() / .parry()
│   ├── WeaponSwitchState           ← brief switch animation
│   └── DeadState
├── HurtBox (Area)
│   └── CollisionShape
├── WeaponManager (Node)            ← tracks active_weapon, routes input to it
│   ├── Bow          (extends WeaponBase)
│   │   ├── HitBox (Area)
│   │   └── ArrowSpawner
│   ├── Dagger       (extends WeaponBase)
│   │   ├── HitBox (Area)
│   │   └── CounterWindow (Timer)   ← opened on perfect dodge
│   ├── Sword        (extends WeaponBase)
│   │   └── HitBox (Area)
│   └── Gauntlet     (extends WeaponBase)
│       └── HitBox (Area)
└── AtmaMask (Node)                 ← mask state, visual, corruption link
    └── MaskSprite
```

### WeaponBase (script — not a node)

Defines the interface all weapons implement. Shared logic (hitbox enable/disable, animation trigger) lives here once.

```
WeaponBase extends Node
│
├── func attack()       → override per weapon
├── func dodge()        → override per weapon  
├── func parry()        → override per weapon
├── func move_modifier()→ override per weapon (e.g. dagger is faster)
│
└── shared base logic:
    ├── enable_hitbox()
    ├── disable_hitbox()
    └── emit hit signals
```

### Per-weapon overrides (what each weapon changes)

| Weapon | attack() | dodge() | parry() |
|---|---|---|---|
| Bow | projectile + combo extender | reposition step | — |
| Dagger | fast multi-hit + bleed | perfect dodge → open CounterWindow | counter strike |
| Sword | moderate dmg | roll | deflect (win vs medium, lose vs huge) |
| Gauntlet | burst + shield break | slow step | block → burst window |

---

## Enemy Scene (base, instanced per type)

```
Enemy (CharacterBody)
├── CollisionShape
├── Sprite / AnimationPlayer
├── NavigationAgent
├── HurtBox (Area)
│   └── CollisionShape
├── HitBox (Area)
│   └── CollisionShape
└── StateMachine (Node)
    ├── IdleState
    ├── ChaseState
    ├── AttackState
    └── DeadState
```

---

## Boss Scene (extends Enemy)

```
Boss (CharacterBody)
├── [all Enemy nodes]
└── PhaseManager (Node)         ← phase 1 / 2 / etc, scales with AtmaManager.corruption
    ├── Phase1
    └── Phase2
```

> Boss difficulty is driven by `AtmaManager.corruption_level` at the time of the fight — the higher the corruption, the harder the fight (per design doc).

---

## Signal Flow

```
Player.took_damage      → HealthBar.update()
Player.died             → AtmaManager.on_player_died()
AtmaManager.revived     → Player.revive()            ← atma sacrifice
AtmaManager.corruption_changed → HUD.update_meter()
AtmaManager.corruption_changed → AtmaMask.update_visual()
AtmaManager.mask_state_changed → Player.apply_mask_stats()
Enemy.died              → EnemySpawner.on_enemy_died()
Boss.phase_changed      → Boss.swap_behavior()
WeaponManager.switched  → WeaponIndicator.update()
```

---

## Mask State Effects (summary for reference)

| State | Trigger | Combat Effect | Ending |
|---|---|---|---|
| Neutral | Default | Normal atk/def/regen | — |
| Consumed | High corruption (many deaths) | Lifesteal, stronger atk, easier | Bad |
| Dominated | Low corruption + Key Mantra | Harder, but purify the Lion | Good |

---

## Notes

- Weapon switching is purely a `WeaponManager` concern — other systems don't care which weapon is active.
- `AtmaManager` is the single source of truth for corruption and mask state. Nothing else tracks it.
- Boss fight difficulty should read from `AtmaManager` once at fight start, not continuously — avoids mid-fight stat jumps.
