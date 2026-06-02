# Node Architecture

High-level Godot scene tree. No scripts here — just structure and signal flow.

---

## Autoloads (Singletons)

```
AtmaManager        ← corruption level, mask state (NEUTRAL / CONSUMED / DOMINATED), ending path
SceneManager       ← scene transitions, loading
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
