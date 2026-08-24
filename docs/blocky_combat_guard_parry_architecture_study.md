# Blocky Sword Battle Royale — Guard / Parry / Reaction Architecture Study

> Reference study based on:
> - Godot 4 Modular Souls-Like Template
> - Game Creator 2 — Melee 2
> - Current Blocky Sword Battle Royale Directional Combat / Guard Runtime work
>
> Goal: extract the most useful combat architecture ideas without replacing the current Three.js combat stack.

---

## 1. Executive Summary

The current combat work should **not** be replaced by Godot or Melee 2.

Instead:

- **Godot Souls-Like Template** provides a very clear minimal implementation of:
  - Guard state
  - Perfect-parry timing
  - Block / Parry resolution
  - Defender reaction
  - Attacker parried reaction
  - Attack interruption

- **Melee 2** provides a more mature data-driven architecture for:
  - Attack phases
  - Skills
  - Shields
  - Reactions
  - Parried reactions
  - Direction / power / cancel timing
  - Callbacks and reaction configuration

- **Blocky Sword Battle Royale** should keep its current advantages:
  - Top / Left / Right attack directions
  - Top / Left / Right guard directions
  - Real blade trajectory
  - Blade contact point
  - Contact metadata
  - Joint bracing
  - Attacker / defender contact coupling

The best architecture is therefore:

```text
Godot event flow
    +
Melee 2 data-driven reactions
    +
Directional Guard
    +
Real blade contact
    +
Procedural joint bracing
```

---

# 2. Godot Souls-Like — Guard → Parry → Attacker Reaction

## 2.1 Guard input

The Godot template enters guard with a very small state machine:

```gdscript
func start_guard():
    slowed = true
    guarding = true
    parry_active = true

    await get_tree().create_timer(parry_window).timeout

    parry_active = false
```

Default behavior:

```text
Guard button pressed

0.0 s
│
├── guarding = true
├── slowed = true
└── parry_active = true
        │
        │ 0.3 sec
        ▼
    parry_active = false

guarding remains true
```

Therefore:

```text
0.00s ─────────────── 0.30s ───────────────────>
        PERFECT PARRY          NORMAL BLOCK
```

This is a very clean minimal design.

However, it is too simple for Directional Combat because the parry timer starts from **input time**, not from the actual guard pose becoming valid.

---

# 3. Better Guard Timing for Blocky Combat

Instead of:

```text
Guard Input
    ↓
Start 0.3 sec timer
```

use animation/contact timing:

```text
Guard Input
    ↓
Guard Enter
    ↓
Guard Ready
    ↓
Perfect Parry Window
    ↓
Guard Hold
```

Example:

```text
0 ms      Guard input

0–70 ms   Raise sword / rotate shoulder / close body opening

70 ms     Guard becomes physically valid

70–220 ms Perfect Parry

220 ms+   Normal Guard
```

Recommended metadata:

```js
guardTimeline: {
  enterStart: 0.00,
  guardReady: 0.07,
  parryStart: 0.07,
  parryEnd: 0.22,
  holdStart: 0.22
}
```

This is more accurate than a generic:

```js
parryWindow = 0.3
```

---

# 4. Godot Guard Animation Architecture

Godot does not treat Guard as a complete independent action.

Instead the AnimationTree continuously blends a Guard layer:

```gdscript
if player_node.guarding:
    guard_value = 1
else:
    guard_value = 0
```

then:

```text
parameters/Guarding/blend_amount
```

Conceptually:

```text
Locomotion
    +
Guard Layer
    ↓
Final Pose
```

This is useful for generic Souls-style shielding.

For Blocky Sword Battle Royale, however, we need:

```text
GuardTop
GuardLeft
GuardRight
```

and potentially:

```text
GuardEnterTop
GuardEnterLeft
GuardEnterRight
```

Therefore Guard should remain a more explicit action/state in our runtime.

---

# 5. Godot Attack Activation Window

The template uses `EquipmentSystem` to enable weapon collision.

Conceptually:

```text
Attack starts
    ↓
wait animationLength × 0.3
    ↓
Weapon collision ON
    ↓
wait animationLength × 0.5
    ↓
Weapon collision OFF
```

Visualized:

```text
0%           30%                         80%       100%
|-------------|===========================|----------|
 Anticipation            Strike             Recovery
```

This is a useful minimal implementation.

But Blocky Combat should **not** copy this percentage-based timing because we already have more precise action/contact metadata.

Recommended:

```text
ActionDefinition

Anticipation
    ↓
StrikeStart
    ↓
Contact Candidate Window
    ↓
StrikeEnd
    ↓
Recovery
```

---

# 6. Godot Hit Detection

Godot uses an Area3D hitbox attached to the weapon.

When the hitbox enters a target:

```gdscript
_hit_body.hit(
    player_node,
    current_equipment.equipment_info
)
```

The combat event is basically:

```text
Attacker
WeaponInfo
Defender
```

Important limitation:

```text
Attacker Sword Hitbox
        ↓
Defender Body
```

It is **not**:

```text
Attacker Blade
        ↓
Defender Blade
        ↓
Weapon-to-Weapon Contact
```

This distinction is extremely important.

Godot's parry is primarily a **logical state check**, not actual blade contact.

---

# 7. Godot Combat Resolver

The central logic is effectively:

```text
Weapon hits Defender
        ↓
Defender.hit(attacker, weapon)
        ↓

hurt cooldown active?
        ↓ no

parry_active?
    │
    ├── YES
    │     ↓
    │ Defender.parry()
    │     +
    │ Attacker.parried()
    │
    └── NO
          ↓
       guarding?
        │
        ├── YES → block()
        │
        └── NO  → damage + hurt
```

Equivalent pseudocode:

```js
function resolveHit(attacker, defender, weapon) {
  if (defender.hurtCooldown) return;

  if (defender.parryActive) {
    defender.parry();
    attacker.parried();
    return;
  }

  if (defender.guarding) {
    defender.block();
    return;
  }

  defender.damage(weapon);
  defender.hurt();
}
```

This is the strongest architectural idea in the Godot implementation:

> Parry is a **two-character reaction event**.

It is not just a defender animation.

---

# 8. Defender Parry Reaction

Successful parry:

```text
Defender.parry()
      ↓
parry_started
      ↓
AnimationTree
      ↓
Parry OneShot
```

Block is similar:

```text
block_started
      ↓
Block OneShot
```

Important design principle:

```text
Combat Logic
    ↓
Combat Event
    ↓
Animation Reaction
```

The animation system does not decide whether the event was a parry.

The combat resolver decides first.

This separation is worth keeping.

---

# 9. Attacker Parried Reaction

This is especially relevant to the current Blocky combat work.

When Defender successfully parries:

```text
Defender
   ↓
Attacker.parried()
```

Enemy side:

```text
Attacker.parried()
      ↓
parried_started
      ↓
AnimationTree
      ↓
Abort current Attack
      ↓
Play "parried" reaction
```

Conceptually:

```text
ATTACK
  │
  │ weapon contact
  ▼
PARRIED
  │
  ├── cancel current attack
  └── play recoil
```

This directly matches the problem currently being solved:

> How should the attacker's animation react after a successful parry?

Godot's answer is architecturally correct:

```text
Interrupt attack
    ↓
Enter attacker reaction
```

But the reaction itself is generic.

---

# 10. Main Limitation of Godot's Parried Reaction

Godot effectively does:

```text
parried()
    ↓
play generic recoil
```

It does not know enough about the collision.

Missing information includes:

```text
Attack direction
Guard direction
Contact point
Contact normal
Blade velocity
Defender blade velocity
Attack power
Contact location on blade
Attack phase
Guard age
Body momentum
```

Therefore the system can only produce:

```text
Generic Parry Recoil
```

Instead of:

```text
Top Parry Recoil
Left Parry Recoil
Right Parry Recoil
```

or physically influenced reactions.

---

# 11. Melee 2 — Why It Is More Mature

Melee 2 treats melee attacks as data-driven Skills.

Conceptually:

```text
Skill
├── Direction
├── Power
├── Animation
├── Anticipation
├── Strike
├── Recovery
├── Striker
├── Motion
├── Conditions
└── Effects
```

This is close to our current `ActionDefinition`.

Melee 2 explicitly separates:

```text
Anticipation
    ↓
Strike
    ↓
Recovery
```

This is preferable to using percentages of animation length.

---

# 12. Melee 2 Shield Architecture

A shield/guard configuration can conceptually contain:

```text
Shield
├── Defense Angle
├── Parry Time
├── Defense / Guard Resource
├── Block Reaction
├── Parry Reaction
└── Break Reaction
```

This means:

```text
Contact
    ↓
Shield Resolver
    ↓
Block / Parry / Break
```

Each result can have its own reaction and callback.

This is much closer to a reusable combat framework.

---

# 13. Melee 2 Reaction Architecture

The strongest Melee 2 concept for our project is the `Reaction`.

A Reaction can be selected using data such as:

```text
Attack Power
Attack Direction
Conditions
```

and can configure:

```text
Animation
Root Motion
Playback Speed
Transition In
Transition Out
Cancel Time
Rotation
Gravity
Avatar Mask
Variants
On Enter
On Exit
```

Instead of:

```text
parried → recoil.fbx
```

the architecture becomes:

```text
ParriedReaction
       ↓
Direction?
       ↓
Power?
       ↓
Character state?
       ↓
Choose recoil variant
```

This is exactly the direction our runtime should move toward.

---

# 14. Defender Reaction vs Attacker Reaction

A successful parry should generate two separate reactions.

```text
                    CONTACT
                       │
                Combat Resolver
                       │
                  result=PARRY
                  /           \
                 /             \
                ▼               ▼
      Defender Reaction    Attacker Reaction
           PARRY              PARRIED
```

These should not share the same definition.

Example:

```js
defenderReaction = {
  type: "parry",
  direction: "top",
  clip: "parry_top"
}

attackerReaction = {
  type: "parried",
  direction: "top",
  clip: "recoil_top"
}
```

---

# 15. Melee 2 Still Does Not Fully Solve Our Goal

Melee 2 supports direction-related reactions and defense angles.

However, it is not a full For-Honor-style directional guard matching system.

Our requirement is:

```text
Attack TOP
Guard TOP
→ valid defense

Attack RIGHT
Guard LEFT
→ guard mismatch
```

Therefore we still need:

```text
AttackDirection
    vs
GuardDirection
```

with:

```text
TOP
LEFT
RIGHT
```

as first-class combat data.

---

# 16. Comparison Table

| Feature | Godot Souls-Like | Melee 2 | Blocky Target |
|---|---|---|---|
| Guard state | Yes | Yes | Yes |
| Parry timing | Fixed timer | Data-driven | Timeline-driven |
| Block | Yes | Yes | Yes |
| Parry | Yes | Yes | Yes |
| Guard break | Limited / no | Yes | Later |
| Attack phases | Rough percentages | Sequencer | Metadata |
| Weapon hitbox | Area3D | Striker | Blade sweep |
| Attack direction | No | Partial / reaction direction | Top / Left / Right |
| Guard direction | No | Defense angle | Top / Left / Right |
| Defender reaction | Yes | Data-driven | Data-driven |
| Attacker recoil | Generic | Parried Reaction | Directional |
| Reaction power | No | Yes | Yes |
| Reaction cancel time | No | Yes | Yes |
| Contact position | No | Supported as hit context | True blade point |
| Weapon-to-weapon contact | No | Not core | Yes |
| Joint bracing | No | No | Yes |
| Contact coupling | No | No | Yes |

---

# 17. Proposed Core Concept: CombatContactEvent

The current runtime should move away from:

```js
guardRuntime.parry();
```

directly deciding everything.

Instead create a complete combat transaction:

```js
CombatContactEvent = {
  attacker,
  defender,

  attackAction,
  attackDirection,

  guardAction,
  guardDirection,

  attackPhase,

  contactPoint,
  contactNormal,

  attackerBladeVelocity,
  defenderBladeVelocity,

  attackPower,

  strikeTime,
  guardAge,

  result
};
```

Recommended result values:

```text
HIT
BLOCK
PARRY
GUARD_FAIL
GUARD_BREAK
MISS
```

---

# 18. Proposed CombatResolver

The resolver should determine the gameplay result.

Example:

```js
function resolveCombatContact(event) {
  if (!event.defender.isGuarding) {
    return "HIT";
  }

  if (!directionMatches(
      event.attackDirection,
      event.guardDirection
  )) {
    return "GUARD_FAIL";
  }

  if (isInsideParryWindow(event.guardAge)) {
    return "PARRY";
  }

  return "BLOCK";
}
```

Later this can also evaluate:

```text
Attack power
Guard stability
Weapon type
Stamina
Blade velocity
Guard break threshold
```

---

# 19. Proposed ReactionDefinition

Inspired by Melee 2:

```js
ReactionDefinition = {
  id: "parried_top_medium",

  result: "PARRY",

  role: "ATTACKER",

  direction: "TOP",

  powerRange: "MEDIUM",

  clip: "recoil_top",

  rootMotion: false,

  transitionIn: 0.05,
  transitionOut: 0.12,

  cancelTime: 0.42,

  procedural: {
    brace: true,
    recoil: true,
    contactCoupling: true
  }
};
```

Defender:

```js
{
  id: "parry_top_defender",

  result: "PARRY",

  role: "DEFENDER",

  direction: "TOP",

  clip: "parry_top",

  procedural: {
    brace: true,
    absorbImpact: true
  }
}
```

---

# 20. Proposed Directional Reactions

## Attacker

```text
parried_top
parried_left
parried_right
```

## Defender

```text
block_top
block_left
block_right

parry_top
parry_left
parry_right
```

These should be selected by the combat event, not hardcoded inside animation code.

---

# 21. Full Recommended Data Flow

```text
                  ATTACKER
                     │
              ActionDefinition
                     │
          Anticipation → Strike
                           │
                       Blade Sweep
                           │
                           ▼
                        CONTACT
                           │
                           ▼
                CombatContactEvent
                           │
             ┌─────────────┴─────────────┐
             │                           │
      attackDirection              guardDirection
            TOP                         TOP
             │                           │
             └─────────────┬─────────────┘
                           ▼
                    CombatResolver
                           │
                timing + direction
                           │
          ┌────────────────┼────────────────┐
          ▼                ▼                ▼
         HIT              BLOCK            PARRY
                                             │
                                ┌────────────┴────────────┐
                                ▼                         ▼
                        Defender Reaction          Attacker Reaction
                           Parry                       Parried
                                │                         │
                         blade brace               recoil response
                         elbow bend                shoulder twist
                         knee absorb               body rotation
                                │                         │
                                └────────────┬────────────┘
                                             ▼
                                      Contact Coupling
                                             │
                                      weapon separation
                                             │
                                      Counter Window
```

---

# 22. Why Animation-Only Fixes Keep Becoming Expensive

The current issue is not simply:

```text
Recoil animation looks bad
```

A convincing parry is a chain:

```text
Weapon contact
    ↓
Momentary coupling
    ↓
Defender absorbs impact
    ↓
Attacker momentum redirects
    ↓
Body joints react
    ↓
Reaction animation completes
```

If all of this responsibility is pushed into a single FBX recoil animation, every new angle requires another manual animation fix.

Instead:

```text
Reaction Animation
        +
Procedural Contact Coupling
        +
Joint Bracing
```

should work together.

Example for a top parry:

```text
Sword contact
    ↓
Wrist stops first
    ↓
Forearm absorbs
    ↓
Shoulder continues slightly
    ↓
Chest rotates
    ↓
Center of mass shifts
    ↓
Rear foot stabilizes
```

That produces a much more believable result than moving the entire character rigidly.

---

# 23. Recommended Development Order

## Phase 1 — CombatContactEvent V1

Centralize:

```text
attacker
defender
attackAction
attackDirection
guardDirection
contactPoint
contactNormal
bladeVelocity
attackPower
guardAge
attackPhase
```

Goal:

> One authoritative description of every combat contact.

---

## Phase 2 — CombatResolver V1

Return:

```text
HIT
BLOCK
PARRY
GUARD_FAIL
```

Later:

```text
GUARD_BREAK
CLASH
DEFLECT
```

---

## Phase 3 — ReactionDefinition V1

Add Melee-2-inspired reaction metadata:

```text
direction
power
clip
rootMotion
cancelTime
transitionIn
transitionOut
procedural modifiers
```

---

## Phase 4 — Directional Attacker Recoil

Implement:

```text
parried_top
parried_left
parried_right
```

No more generic recoil.

---

## Phase 5 — Directional Guard Reactions

Implement:

```text
block_top
block_left
block_right

parry_top
parry_left
parry_right
```

---

## Phase 6 — Contact Coupling / Joint Bracing

After the reaction architecture is stable:

```text
Contact Point
    ↓
Temporary Blade Constraint
    ↓
Joint Bracing
    ↓
Body Reaction
    ↓
Blade Release
```

This should remain procedural and layered on top of animation.

---

# 24. Recommended Ownership Boundaries

## ActionDefinition owns

```text
Attack animation
Attack direction
Attack phases
Strike window
Contact metadata
Attack power
```

## GuardDefinition owns

```text
Guard direction
Guard enter timeline
Parry window
Guard hold
Guard stability
```

## CombatContactEvent owns

```text
What physically happened
Who contacted whom
Where
When
In what direction
With what velocity / power
```

## CombatResolver owns

```text
HIT / BLOCK / PARRY / GUARD_FAIL
```

## ReactionDefinition owns

```text
Which reaction plays
How long it locks control
Root motion
Blend timing
Reaction variants
```

## Contact Coupling owns

```text
Temporary physical relationship
between attacker weapon,
defender weapon,
and both character rigs
```

---

# 25. Key Architectural Principle

The most important rule:

> **Animation must not decide combat truth.**

Instead:

```text
Contact
    ↓
Combat Resolver
    ↓
Result
    ↓
Reaction Selection
    ↓
Animation + Procedural Response
```

Never:

```text
Animation looks like a parry
    ↓
therefore result = parry
```

Combat logic and animation should remain separated.

---

# 26. Final Recommendation

Do not replace the existing Three.js combat work with Godot or Melee 2.

Use them as references:

```text
Godot Souls-Like
→ Minimal event chain reference

Melee 2
→ Mature data-driven combat architecture reference

Blocky Sword Battle Royale
→ Directional weapon contact + procedural reaction system
```

The target architecture should become:

```text
ActionDefinition
      +
GuardDefinition
      ↓
Blade Contact
      ↓
CombatContactEvent
      ↓
CombatResolver
      ↓
ReactionDefinition
      ↓
Attacker / Defender Reactions
      ↓
Contact Coupling
      ↓
Joint Bracing
      ↓
Recovery / Counter Window
```

This prevents future animation work from repeatedly modifying combat logic.

It also allows new animation packs — KayKit, UAL, converted Skyrim animations, or custom authored clips — to be swapped through `ReactionDefinition.clip` without rewriting the entire Guard Runtime.

---

# 27. Practical Next Step

Before continuing detailed animation polishing:

```text
1. CombatContactEvent V1
2. CombatResolver V1
3. ReactionDefinition V1
```

should be implemented first.

Only after those are stable should work continue on:

```text
Directional Recoil
Directional Block Hit
Directional Parry
Joint Bracing
Contact Coupling
```

This gives the project a reusable PvP melee foundation rather than continuing to fix individual animations one by one.
