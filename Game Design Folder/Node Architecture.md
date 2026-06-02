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

Top-level StateMachine handles character-wide states only. Each weapon owns its own combat sub-states — no shared AttackState or DodgeState that checks which weapon is active.

```
Player (CharacterBody)
├── CollisionShape
├── Sprite / AnimationPlayer
├── StateMachine (Node)             ← character-level states only
│   ├── IdleState
│   ├── MoveState
│   ├── WeaponActionState           ← delegates to active weapon's sub-SM
│   ├── WeaponSwitchState           ← brief switch animation
│   └── DeadState
├── HurtBox (Area)
│   └── CollisionShape
├── WeaponManager (Node)            ← tracks active weapon, routes input
│   ├── Bow
│   │   ├── HitBox (Area)
│   │   ├── ArrowSpawner
│   │   └── StateMachine            ← bow-specific states
│   │       ├── AttackState         (projectile, low dmg, combo extender)
│   │       ├── AimState            (hold to aim, movement allowed)
│   │       └── KnockState          (knock flying enemy follow-up)
│   ├── Dagger
│   │   ├── HitBox (Area)
│   │   └── StateMachine            ← dagger-specific states
│   │       ├── AttackState         (fast, bleed on hit)
│   │       └── DodgeState          (perfect dodge → opens CounterWindow)
│   │           └── CounterWindow   (Timer — counter attack if hit in window)
│   ├── Sword
│   │   ├── HitBox (Area)
│   │   └── StateMachine            ← sword-specific states
│   │       ├── AttackState         (moderate dmg, all-rounder)
│   │       └── DeflectState        (parry — win vs medium, lose vs huge)
│   └── Gauntlet
│       ├── HitBox (Area)
│       └── StateMachine            ← gauntlet-specific states
│           ├── AttackState         (burst dmg, shield destroyer)
│           └── BlockState          (deflect, absorb — then burst window)
└── AtmaMask (Node)                 ← mask state, visual, corruption link
    └── MaskSprite
```

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
