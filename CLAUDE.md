# Mastery-System Pixel Dungeon

## 1. Project Vision

**Mastery-System Pixel Dungeon** is a deep refactoring project based on Shattered Pixel Dungeon. The core goal is to build a **web-playable** roguelike game featuring **Dark Souls-style invasion PvP**.

### Core Vision
- **Web-First**: Players can play directly in the browser without installation
- **Invasion PvP**: Dark Souls-like "invasion" mechanic where other players can enter your world for combat
- **Weapon Mastery System**: Unique tiered Weapon Mastery system that deepens combat strategy
- **Deterministic Engine**: Guarantees identical outputs for identical inputs - the foundation of multiplayer synchronization

### Multiplayer Design Philosophy
This is NOT a co-op adventure mode. This is **adversarial invasion**:
- Player A is progressing through the dungeon
- Player B can "invade" Player A's world
- Invader and host engage in PvP combat on the same map
- Defeating an invader grants rewards; being defeated incurs penalties

---

## 2. Tech Stack

| Area | Technology |
|------|------------|
| Language | Java 11 |
| Build Tool | Gradle (multi-module) |
| Game Framework | LibGDX 1.13.6-SNAPSHOT |
| Web Compilation | GWT (Google Web Toolkit) |
| Networking | WebSocket (planned) |
| Platforms | Desktop, Android, iOS, **Web** |

---

## 3. Project Structure

```
mastery-pixel-dungeon/
├── engine/                    # 🎯 Core Engine (LibGDX-free, GWT-safe)
│   ├── ability/               # Skill/ability system
│   ├── actor/                 # Actors, heroes, mobs
│   │   └── buff/              # Buff/status effect system
│   ├── combat/                # Combat formulas
│   ├── command/               # GameCommand/GameEvent pipeline
│   ├── dungeon/               # Dungeons, terrain, FOV
│   ├── item/                  # Item system
│   ├── replay/                # Replay system, state hashing
│   ├── scheduling/            # Turn scheduler
│   └── serialization/         # Save/load, DTOs
│
├── network/                   # 🌐 Network Layer (planned)
│   ├── protocol/              # Message protocol definitions
│   ├── server/                # Game server logic
│   └── client/                # Network client logic
│
├── client-common/             # Shared client logic
├── client-desktop/            # Desktop platform
├── client-android/            # Android platform
├── client-web/                # 🌐 Web platform (GWT)
│
├── mod-loader/                # Mod loader
├── api/                       # Public API
│
└── docs/                      # Documentation
    ├── progress.md            # 📊 Detailed development progress
    ├── getting-started-*.md   # Platform build guides
    └── recommended-changes.md # Architecture recommendations
```

---

## 4. Architecture Rules (Strictly Enforced)

### 4.1 Engine Module Constraints
```
✅ ALLOWED                       ❌ FORBIDDEN
─────────────────────────────────────────────────────
Pure Java logic                  LibGDX dependencies
DataOutputStream serialization   Java Reflection
Explicit factory registration    Class.forName() etc.
Fixed-seed random numbers        System.currentTimeMillis()
Immutable data structures        Shared mutable state
```

### 4.2 Determinism Rules
- **Same Seed + Same Commands = Same Outcome**
- All random numbers must come from SeededRandom
- No iteration over non-deterministic collections (e.g., HashMap iteration order)
- No system time as game logic input

### 4.3 Serialization Rules
- All game state must be serializable
- Use binary format (DataInputStream/DataOutputStream)
- Every new class must have a corresponding Snapshot DTO
- Must pass round-trip tests

### 4.4 GWT Compatibility
- No Reflection
- No Class metadata inspection
- Use explicit Registry pattern instead of reflection factories
- Avoid certain Java 8+ APIs (check GWT compatibility list)

---

## 5. Current Progress

### Completed ✅
| Phase | Content | Status |
|-------|---------|--------|
| 1-3 | Core engine architecture (Command/Event, movement, combat, FOV) | ✅ Complete |
| 4 | Deterministic replay system (serialization, state hashing, replay verification) | ✅ Complete |
| 5 | Buff system (framework, core buffs, stacking, removal, death integration) | ✅ Complete |
| 6.1-6.4 | Ability system (framework, time cost, cooldowns, usage limits) | ✅ Complete |
| 7.1-7.3 | Item system foundations + Resource system (Mana/Stamina) | ✅ Complete |

### In Progress 🚧
| Phase | Content | Status |
|-------|---------|--------|
| 7.4+ | Equipment slots, item stacking, expanded item library | 🚧 Pending |
| 8 | Procedural dungeon generation | 📋 Planned |
| 12 | **Multiplayer sync (Invasion PvP)** | 📋 Planned |

### Detailed Progress
👉 See `docs/progress.md` for full Phase details and commit history

---

## 6. Multiplayer Architecture (Planned)

### 6.1 Invasion Mechanic Design

```
┌─────────────────────────────────────────────────────────────┐
│                     Matchmaking Server                       │
│  - Manages list of "invadable" games                         │
│  - Pairs invaders with hosts                                 │
└─────────────────────────────────────────────────────────────┘
                              │
          ┌───────────────────┼───────────────────┐
          ▼                   ▼                   ▼
    ┌──────────┐        ┌──────────┐        ┌──────────┐
    │ Player A │        │ Player B │        │ Player C │
    │ (Host)   │◄──────►│(Invader) │        │ (Solo)   │
    │ Dungeon  │ Invade │ Ready to │        │ Offline  │
    │ crawling │        │ invade   │        │ mode     │
    └──────────┘        └──────────┘        └──────────┘
```

### 6.2 Synchronization Strategy

Using **Lockstep + Command Replication** model:

1. **Host Authoritative**: The invaded player's game instance is the authority
2. **Command Sync**: Invader's actions sent as GameCommands to Host
3. **State Verification**: Periodic StateHash comparison to detect desync
4. **Rollback**: On desync, invader rolls back to Host state

### 6.3 Network Message Types (Planned)

```java
// Connection establishment
InvasionRequest      // Request to invade a player
InvasionAccepted     // Invasion request accepted
InvasionRejected     // Invasion request rejected

// Game synchronization
CommandMessage       // Player action commands
StateSnapshot        // Full state snapshot
StateHash            // State hash (lightweight verification)
SyncRequest          // Request full sync

// Battle results
InvasionResult       // Invasion outcome (win/lose/flee)
RewardGrant          // Reward distribution
```

---

## 7. Common Commands

```bash
# Build and run
./gradlew desktop:debug          # Run Desktop debug build
./gradlew desktop:release        # Build Desktop release JAR
./gradlew android:assembleDebug  # Build Android APK
./gradlew html:superDev          # Run Web dev server (GWT)

# Testing
./gradlew engine:test            # Run Engine unit tests
./gradlew test                   # Run all tests

# Code quality
./gradlew checkstyle             # Code style check
./gradlew spotbugs               # Static analysis
```

---

## 8. AI Development Guidelines

### 8.1 Development Principles

1. **Determinism First**: Any new feature must guarantee determinism, tested with fixed seeds
2. **Serialization Verification**: New state must implement serialization and pass round-trip tests
3. **GWT Compatible**: Engine code must not use Reflection or LibGDX
4. **Replay Testing**: Important features require Replay Hash verification

### 8.2 Commit Convention

```
Phase X.Y: <short description>

<detailed explanation>

- Added: <new features>
- Changed: <modifications>
- Tests: <test coverage>
```

### 8.3 Test Requirements

Each Phase must include:
- [ ] Unit tests (basic functionality)
- [ ] Fixed-seed tests (determinism)
- [ ] Save/Load tests (serialization)
- [ ] Replay Hash tests (replay verification)

### 8.4 Branch Strategy

- Development branch: `claude/refactor-dungeon-engine-*`
- Feature branches: `feature/<phase>-<description>`
- Main branch: Stable releases

---

## 9. Version Info

| Item | Value |
|------|-------|
| Base Version | Shattered Pixel Dungeon 3.2.5 |
| Package | `com.shatteredpixel.shatteredpixeldungeon` |
| Version Code | 877 |
| Java Target | 11 |
| Android Min SDK | 21 |
| Android Target SDK | 35 |

---

## 10. Key Documents

| Document | Purpose |
|----------|---------|
| `docs/progress.md` | 📊 Complete development progress tracking (Phase details) |
| `docs/getting-started-desktop.md` | Desktop build guide |
| `docs/getting-started-android.md` | Android build guide |
| `docs/recommended-changes.md` | Architecture recommendations |

---

## 11. Next Priorities

### Short-term (Complete Phase 7)
1. Phase 7.4: Equipment system (weapons, armor, rings)
2. Phase 7.5: Item stacking and inventory management
3. Phase 7.6: Expanded item library

### Mid-term (Phase 8-9)
1. Phase 8: Procedural dungeon generation
2. Phase 9: Advanced AI behaviors (pathfinding, tactics)

### Long-term (Phase 12+)
1. Phase 12: **Multiplayer synchronization infrastructure**
2. Phase 13: **Invasion PvP implementation**
3. Phase 14: **Web client optimization**

---

## 12. References

- [Shattered Pixel Dungeon (Original Project)](https://github.com/00-Evan/shattered-pixel-dungeon)
- [LibGDX Framework](https://libgdx.com/)
- [GWT Project](http://www.gwtproject.org/)
- [pixel-dungeon-gdx (Web Port Reference)](https://github.com/gnojus/pixel-dungeon-gdx)
- [pixel-dungeon-multiplayer (Multiplayer Reference)](https://github.com/Nikita22007/pixel-dungeon-multiplayer)

---

*Last Updated: 2025-12-01*
*Development Branch: `claude/refactor-dungeon-engine-01CB2n9xqFqi65FqNMx1Nhtb`*
