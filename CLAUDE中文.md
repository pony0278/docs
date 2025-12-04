# Mastery-System Pixel Dungeon

## 1. Project Vision

**Mastery-System Pixel Dungeon** 是一個基於 Shattered Pixel Dungeon 的深度重構專案，核心目標是打造一個**支援網頁遊玩**、具備 **Dark Souls 風格入侵式 PvP** 的 roguelike 遊戲。

### 核心願景
- **Web-First**: 玩家可直接在瀏覽器中遊玩，無需安裝
- **入侵式 PvP**: 類似 Dark Souls 的「入侵」機制，其他玩家可以進入你的世界進行對戰
- **武器精通系統**: 獨特的 Weapon Mastery 分層系統，深化戰鬥策略
- **確定性引擎**: 保證相同輸入產生相同結果，這是多人同步的基石

### 多人對戰設計理念
這不是合作冒險模式，而是**對抗式入侵**：
- 玩家 A 正在進行地牢冒險
- 玩家 B 可以「入侵」玩家 A 的世界
- 入侵者與被入侵者在同一地圖中進行 PvP 對戰
- 擊敗入侵者可獲得獎勵，被擊敗則承受懲罰

---

## 2. Tech Stack

| 領域 | 技術 |
|------|------|
| 語言 | Java 11 |
| 建構工具 | Gradle (multi-module) |
| 遊戲框架 | LibGDX 1.13.6-SNAPSHOT |
| 網頁編譯 | GWT (Google Web Toolkit) |
| 網路通訊 | WebSocket (規劃中) |
| 平台支援 | Desktop, Android, iOS, **Web** |

---

## 3. Project Structure

```
mastery-pixel-dungeon/
├── engine/                    # 🎯 核心引擎 (LibGDX-free, GWT-safe)
│   ├── ability/               # 技能系統
│   ├── actor/                 # 角色、英雄、怪物
│   │   └── buff/              # Buff/狀態系統
│   ├── combat/                # 戰鬥公式
│   ├── command/               # GameCommand/GameEvent 管線
│   ├── dungeon/               # 地城、地形、FOV
│   ├── item/                  # 道具系統
│   ├── replay/                # 回放系統、狀態雜湊
│   ├── scheduling/            # 回合排程器
│   └── serialization/         # 存檔/讀檔、DTO
│
├── network/                   # 🌐 網路層 (規劃中)
│   ├── protocol/              # 訊息協定定義
│   ├── server/                # 遊戲伺服器邏輯
│   └── client/                # 網路客戶端邏輯
│
├── client-common/             # 共用客戶端邏輯
├── client-desktop/            # Desktop 平台
├── client-android/            # Android 平台
├── client-web/                # 🌐 Web 平台 (GWT)
│
├── mod-loader/                # Mod 載入器
├── api/                       # 公開 API
│
└── docs/                      # 文件
    ├── progress.md            # 📊 詳細開發進度追蹤
    ├── getting-started-*.md   # 平台建置指南
    └── recommended-changes.md # 架構建議
```

---

## 4. Architecture Rules (嚴格遵守)

### 4.1 Engine 模組限制
```
✅ 允許                          ❌ 禁止
─────────────────────────────────────────────────────
純 Java 邏輯                     LibGDX 依賴
DataOutputStream 序列化          Java Reflection
明確的 Factory 註冊              Class.forName() 等
固定種子的隨機數                  System.currentTimeMillis()
不可變資料結構優先                共享可變狀態
```

### 4.2 確定性規則 (Determinism)
- **Same Seed + Same Commands = Same Outcome**
- 所有隨機數必須來自 SeededRandom
- 禁止依賴執行順序不確定的集合 (如 HashMap 迭代)
- 禁止使用系統時間作為遊戲邏輯輸入

### 4.3 序列化規則
- 所有遊戲狀態必須可序列化
- 使用 Binary 格式 (DataInputStream/DataOutputStream)
- 每個新類別必須有對應的 Snapshot DTO
- 必須通過 Round-trip 測試

### 4.4 GWT 相容性
- 無 Reflection
- 無 Class metadata 檢查
- 使用明確的 Registry 模式替代反射工廠
- 避免 Java 8+ 的部分 API (檢查 GWT 相容清單)

---

## 5. Current Progress

### 已完成 ✅
| Phase | 內容 | 狀態 |
|-------|------|------|
| 1-3 | 核心引擎架構 (Command/Event, 移動, 戰鬥, FOV) | ✅ 完成 |
| 4 | 確定性回放系統 (序列化, 狀態雜湊, 回放驗證) | ✅ 完成 |
| 5 | Buff 系統 (框架, 核心 Buff, 堆疊, 移除, 死亡整合) | ✅ 完成 |
| 6.1-6.4 | 技能系統 (框架, 時間消耗, 冷卻, 使用限制) | ✅ 完成 |
| 7.1-7.3 | 道具系統基礎 + 資源系統 (Mana/Stamina) | ✅ 完成 |

### 進行中 🚧
| Phase | 內容 | 狀態 |
|-------|------|------|
| 7.4+ | 裝備欄位、道具堆疊、擴展道具庫 | 🚧 待開始 |
| 8 | 程序化地城生成 | 📋 規劃中 |
| 12 | **多人同步 (入侵式 PvP)** | 📋 規劃中 |

### 詳細進度
👉 參見 `docs/progress.md` 取得完整的 Phase 細節與 Commit 記錄

---

## 6. Multiplayer Architecture (規劃)

### 6.1 入侵機制設計

```
┌─────────────────────────────────────────────────────────────┐
│                     Matchmaking Server                       │
│  - 管理「可入侵」的遊戲列表                                    │
│  - 配對入侵者與被入侵者                                        │
└─────────────────────────────────────────────────────────────┘
                              │
          ┌───────────────────┼───────────────────┐
          ▼                   ▼                   ▼
    ┌──────────┐        ┌──────────┐        ┌──────────┐
    │ Player A │        │ Player B │        │ Player C │
    │ (Host)   │◄──────►│(Invader) │        │ (Solo)   │
    │ 地城冒險  │  入侵   │ 準備入侵  │        │ 離線模式 │
    └──────────┘        └──────────┘        └──────────┘
```

### 6.2 同步策略

採用 **Lockstep + Command Replication** 模式：

1. **Host 權威**: 被入侵者的遊戲實例為權威來源
2. **Command 同步**: 入侵者的操作作為 GameCommand 傳送給 Host
3. **State 驗證**: 定期進行 StateHash 比對，檢測 Desync
4. **Rollback**: 發生 Desync 時，入侵者回滾到 Host 狀態

### 6.3 網路訊息類型 (規劃)

```java
// 連線建立
InvasionRequest      // 請求入侵某玩家
InvasionAccepted     // 入侵請求被接受
InvasionRejected     // 入侵請求被拒絕

// 遊戲同步
CommandMessage       // 玩家操作指令
StateSnapshot        // 完整狀態快照
StateHash            // 狀態雜湊 (輕量驗證)
SyncRequest          // 請求完整同步

// 對戰結果
InvasionResult       // 入侵結果 (勝/敗/逃跑)
RewardGrant          // 獎勵發放
```

---

## 7. Common Commands

```bash
# 建置與執行
./gradlew desktop:debug          # 執行 Desktop 除錯版
./gradlew desktop:release        # 建置 Desktop 發布版
./gradlew android:assembleDebug  # 建置 Android APK
./gradlew html:superDev          # 執行 Web 開發伺服器 (GWT)

# 測試
./gradlew engine:test            # 執行 Engine 單元測試
./gradlew test                   # 執行所有測試

# 程式碼品質
./gradlew checkstyle             # 程式碼風格檢查
./gradlew spotbugs               # 靜態分析
```

---

## 8. AI Development Guidelines

### 8.1 開發原則

1. **確定性優先**: 任何新功能必須保證確定性，使用固定種子測試
2. **序列化驗證**: 新增狀態必須實作序列化並通過 Round-trip 測試
3. **GWT 相容**: Engine 程式碼禁用 Reflection，禁用 LibGDX
4. **回放測試**: 重要功能需要 Replay Hash 驗證

### 8.2 Commit 規範

```
Phase X.Y: <簡短描述>

<詳細說明>

- 新增: <新功能>
- 修改: <變更內容>
- 測試: <測試涵蓋範圍>
```

### 8.3 測試要求

每個 Phase 必須包含：
- [ ] 單元測試 (基本功能)
- [ ] 固定種子測試 (確定性)
- [ ] Save/Load 測試 (序列化)
- [ ] Replay Hash 測試 (回放驗證)

### 8.4 分支策略

- 開發分支: `claude/refactor-dungeon-engine-*`
- 功能分支: `feature/<phase>-<description>`
- 主分支: 穩定版本

---

## 9. Version Info

| 項目 | 值 |
|------|-----|
| 基底版本 | Shattered Pixel Dungeon 3.2.5 |
| Package | `com.shatteredpixel.shatteredpixeldungeon` |
| Version Code | 877 |
| Java Target | 11 |
| Android Min SDK | 21 |
| Android Target SDK | 35 |

---

## 10. Key Documents

| 文件 | 用途 |
|------|------|
| `docs/progress.md` | 📊 完整開發進度追蹤 (Phase 細節) |
| `docs/getting-started-desktop.md` | Desktop 建置指南 |
| `docs/getting-started-android.md` | Android 建置指南 |
| `docs/recommended-changes.md` | 架構建議 |

---

## 11. Next Priorities

### 短期 (Phase 7 完成)
1. Phase 7.4: 裝備系統 (武器、盔甲、戒指)
2. Phase 7.5: 道具堆疊與背包管理
3. Phase 7.6: 擴展道具庫

### 中期 (Phase 8-9)
1. Phase 8: 程序化地城生成
2. Phase 9: 進階 AI 行為 (尋路、戰術)

### 長期 (Phase 12+)
1. Phase 12: **多人同步基礎架構**
2. Phase 13: **入侵式 PvP 實作**
3. Phase 14: **Web 客戶端最佳化**

---

## 12. References

- [Shattered Pixel Dungeon (原始專案)](https://github.com/00-Evan/shattered-pixel-dungeon)
- [LibGDX Framework](https://libgdx.com/)
- [GWT Project](http://www.gwtproject.org/)
- [pixel-dungeon-gdx (Web 移植參考)](https://github.com/gnojus/pixel-dungeon-gdx)
- [pixel-dungeon-multiplayer (多人參考)](https://github.com/Nikita22007/pixel-dungeon-multiplayer)

---

*Last Updated: 2025-12-01*
*Development Branch: `claude/refactor-dungeon-engine-01CB2n9xqFqi65FqNMx1Nhtb`*
