# Move Data Schema (PUNCH STUDIO ↔ Engine 契約)

> **目的**：定義招式資料（pose + timeline + 比例）的交換格式，作為 **PUNCH STUDIO 編輯器** 與 **`engine/` 戰鬥引擎** 之間的正式契約。
>
> **定位**：目前「先自用」服務 Mastery PD，但格式刻意設計成**引擎中立**，保留未來外推成獨立工具的可能。
>
> **狀態**：`version: 4`（對應 PUNCH STUDIO 的 `STATE_VERSION = 4`）。

---

## 1. 設計原則

1. **引擎是唯一真相（Engine is source of truth）**
   這份檔案只描述 **keyframe 參數**。插值、easing、lag、播放一律由 `engine/` 計算。
   PUNCH STUDIO 的 3D 預覽是**非權威**的參考，**不得**參與多人同步。
2. **確定性（Determinism）**
   消費端的插值/easing 必須是確定性的（fixed-point 或嚴格定義的 float 行為），
   才能滿足 Mastery PD 的 lockstep PvP。詳見 §6。
3. **`poseKeys` 是權威軸列表**
   消費端**應讀取檔案內的 `poseKeys` 陣列**來決定軸的順序與數量，
   **不要** hardcode（v4 為 47 軸；v3=41、v2=39，以 `poseKeys.length` 為準）。
4. **向前相容**：未知欄位應被忽略而非報錯；缺漏欄位以預設值補齊（見各節）。

---

## 2. 頂層結構

```jsonc
{
  "version": 2,                 // 格式版本（整數）。消費端必須檢查。
  "createdBy": "PUNCH STUDIO",  // 產生者標記（資訊用途）
  "poseKeys": [ "root_y", ... ],// 權威軸名陣列（v4 = 47 軸，順序固定，見 §3）
  "seq":    [ /* TimelineKey */ ],  // 有序時間軸（= 格鬥 frame data），見 §4
  "phases": { "<keyName>": { /* Pose */ } }, // 每個 key 一組 pose，見 §3
  "lags":   { "aL":0, "aR":0.2, "lL":0, "lR":0.1 }, // per-limb 鞭打延遲，見 §5
  "dim":    { /* 角色比例 */ }   // 結構比例（retarget 用），見 §5
}
```

- `phases` 的 key 集合**必須**等於 `seq` 內所有 `name`。
- 基準幀率 **`REF_FPS = 60`**，所有 `frame` / `frames` 以此為單位。

---

## 3. Pose（每個 timeline key 一組）

`phases.<keyName>` 是一個物件，鍵為 `poseKeys` 中的軸名，值為數字。
**scale 類軸（`body_scale`、任何 `*_scale`）預設為 `1`；其餘預設 `0`。**
缺漏軸由消費端補預設值。

### v4 的 47 條軸（順序 = `poseKeys`）

| 群組 | 軸 | 說明 | 範圍 | 單位 |
|------|----|------|------|------|
| **ROOT** | `root_y` | 軀幹 Y（twist/側轉） | −120…120 | ° |
| | `root_x` | 軀幹 X（pitch/俯仰） | −30…30 | ° |
| | `root_py` | 軀幹升降（jump/離地） | −0.5…0.4 | u |
| | `root_pz` | 軀幹 Z（lunge 前後） | −0.4…0.6 | u |
| | `sq` | squash/stretch | −0.4…0.4 | — |
| | `body_scale` | 身體縮放 | 0.5…1.5 | × |
| | `squat` | 蹲下（自動踩地） | 0…80 | ° |
| **HEAD** | `head_y` | 頭 Y（左右轉） | −90…90 | ° |
| | `head_x` | 頭 X（俯仰） | −45…45 | ° |
| | `head_pz` | 頭 Z 位置 | −0.2…0.5 | u |
| **SPINE/PELVIS** | `spine_x` | 脊椎 X（前傾，正=前） | −30…60 | ° |
| | `spine_y` | 脊椎 Y（上半身扭轉） | −60…60 | ° |
| | `pelvis_y` | 骨盆 Y（下半身碾地轉） | −60…60 | ° |
| **ARM L**（前手） | `aL_sx` | 肩 X（上下/前後） | −180…60 | ° |
| | `aL_sy` | 肩 Y（左右橫掃） | −90…90 | ° |
| | `aL_sz` | 肩 Z（外展，正=往外） | −45…170 | ° |
| | `aL_ex` | 肘（0=直 / 180=折） | −20…160 | ° |
| | `aL_idle` | → IDLE 比例（1=強制垂下） | 0…1 | — |
| | `aL_scale` | 前臂/拳頭縮放（命中放大） | 0.5…2.5 | × |
| | `aL_wx` | 腕 X（屈伸/勾腕，v4 新增） | −90…90 | ° |
| | `aL_wy` | 腕 Y（沿前臂軸扭轉/旋前旋後，v4 新增） | −90…90 | ° |
| **ARM R**（後手） | `aR_sx` `aR_sy` `aR_sz` `aR_ex` `aR_idle` `aR_scale` `aR_wx` `aR_wy` | 同 ARM L | 同上 | |
| **LEG L**（前腿） | `lL_hx` | 髖 X（前後擺） | −60…60 | ° |
| | `lL_hy` | 髖 Y（外旋/整條腿轉，正=外） | −150…150 | ° |
| | `lL_hz` | 髖 Z（橫向張開/劈腿，正=外） | −60…120 | ° |
| | `lL_kx` | 膝（0=直 / 90=蹲，hinge 僅 X） | −20…90 | ° |
| | `lL_ax` | 腳踝 X 微調 | −60…60 | ° |
| | `lL_idle` | → IDLE 比例（1=強制直立） | 0…1 | — |
| | `lL_scale` | 小腿/腳掌縮放 | 0.5…2.5 | × |
| | `lL_contact` | 腳掌接觸鎖（0=平踩 / 1=墊腳抬跟 / 2=抬起離地） | 0 / 1 / 2 | enum |
| | `lL_ty` | 腳尖朝向（踝 Y，正=外八，v4 新增；可獨立於髖瞄準腳尖） | −120…120 | ° |
| **LEG R**（後腿） | `lR_hx` `lR_hy` `lR_hz` `lR_kx` `lR_ax` `lR_idle` `lR_scale` `lR_contact` `lR_ty` | 同 LEG L | 同上 | |

> 範圍為編輯器 slider 的軟限制；消費端**建議**依此 clamp 作為合法性檢查（也可當關節角度上限，輔助 §「景深歧義」的 Solve 約束）。
> 單位：`°`=度，`u`=世界單位（相對 `dim`），`×`=倍率，`enum`=離散整數。
>
> **腳掌接觸鎖（`lL_contact` / `lR_contact`，v3 新增）**：定義該腳與地面的接觸狀態，消費端據此決定「踩地」與「重心」。
> - `0 平踩`：整個腳掌貼地，當地面錨點。
> - `1 墊腳`：抬腳跟、以腳尖為支點（拳擊後腳碾地/重心轉移），仍當地面錨點（錨在腳尖）。
> - `2 抬起`：腳離地，**不**當地面錨點 → 身體高度由另一支撐腳決定（重心落在支撐腳）。
> 規則：角色的垂直高度 = 把「所有非抬起腳」的最低接觸點對到地面 Y=0（再加 `root_py` 升降）。
> 雙腳皆抬起時退化為「雙腳最低點」以免角色飄走。

---

## 4. Timeline（`seq`）= 格鬥 frame data

`seq` 是**有序陣列**，定義播放順序與時間。語意上對應格鬥遊戲的 **startup → active → recovery**。

```jsonc
{
  "name": "strike",     // key 識別名（唯一）。seq[0] 必為 "idle"
  "frame": 13,          // 絕對幀（@60fps）。seq[0].frame = 0
  "frames": 3,          // 段長 = 此 key.frame − 前一 key.frame（衍生值）
  "ease": "in",         // 'out' | 'in' | 'lin'（緩動；定義見 §6）
  "impact": false,      // true = 命中段（active frames）：無 lag、可掛 hitbox/傷害
  "tag": "strike",      // 語意標籤（見下）
  "returnFrames": 10    // 僅 seq[0]（idle）有效：loop 收尾回 idle 的時長
}
```

**約束**
- `seq[0]` 永遠是 `idle`，`frame = 0`，`impact = false`。
- `frame` 嚴格遞增；每個 key 的 `frame` 唯一。
- `frames` 為衍生值（= 與前一 key 的差）。以 `frame` 為主、`frames` 為輔；
  兩者衝突時以 `frame` 為準（編輯器的 `repairTimeline` 即如此修復）。
- `tag` ∈ `idle | anti | strike | impact | follow | recover | custom`（純語意/UI，不影響播放）。

**玩法層掛載點（未來）**：`impact: true` 的段 = active frames，
是引擎掛 **hitbox / 傷害 / 受擊反應 / cancel window** 的位置。本格式 v2 尚未含這些欄位，
未來新增時應放在 `seq` entry 下（例如 `hit: { box, damage, ... }`）並升版本號。

---

## 5. `lags` 與 `dim`

### `lags`（per-limb 鞭打延遲）
四肢相對主時間軸的延遲，製造鞭打感。`0` = 與軀幹同步。

| 鍵 | 說明 | 預設 |
|----|------|------|
| `aL` | 左手 lag | 0.0 |
| `aR` | 右手 lag | 0.20 |
| `lL` | 左腿 lag | 0.0 |
| `lR` | 右腿 lag | 0.10 |

> `impact` 段不套用 lag（命中要齊）。確切的 lag→插值對應由引擎定義（§6）。

### `dim`（角色結構比例 / retarget）
與 per-phase pose **無關**的骨架尺寸。retarget 到不同體型時，
pose 的角度軸可沿用，`*_py/*_pz/*_scale` 等位置/倍率軸需依 `dim` 重新解讀。

| 鍵 | 預設 | | 鍵 | 預設 |
|----|------|---|----|------|
| `headSize` | 0.84 | | `armLenL` / `armLenR` | 1.00 |
| `bodyH` / `bodyW` / `bodyD` | 0.78 / 0.86 / 0.56 | | `legUpper` / `legLower` / `legThick` | 0.34 / 0.45 / 1.23 |
| `armUpper` / `armLower` / `armThick` | 0.25 / 0.30 / 0.90 | | `fist` / `shoe` | 0.71 / 1.11 |

---

## 6. 確定性契約（消費端必須遵守）

這是 PvP 同步的關鍵，**工具端的 JS 預覽不算數**。引擎實作時必須鎖定：

1. **Easing 定義**（三種，輸入 `t∈[0,1]`，必須與雙方一致）
   - `lin`：`t`
   - `in`（ease-in）：建議 `t*t`
   - `out`（ease-out）：建議 `1-(1-t)²`
   （上述為建議式；最終以引擎實作為準，並回寫此文件。工具預覽須對齊引擎，而非反向。）
2. **取樣**：在整數幀（@60fps）上求值；段內以該 key 的 `ease` 對 `[prevKey, thisKey]` 插值。
3. **lag**：四肢的相位延遲量化方式（幀或比例）由引擎定義，避免 JS/Java 浮點分歧。
4. **GWT/Java 浮點**：跨平台（Desktop JVM ↔ GWT）須產生一致結果；
   有疑慮的軸建議改 fixed-point 或在量化後存整數。詳見 `CLAUDE.md` §4.2 確定性規則。
5. **序列化**：引擎側每個對應結構需有 Snapshot DTO 並通過 round-trip 測試（`CLAUDE.md` §4.3）。

---

## 7. 版本策略

- `version` 為**整數**，破壞性變更（軸增減、語意改變）才 +1，並在本文件記錄遷移。
- **新增**欄位（如 `seq[].hit`）若可被舊消費端安全忽略，**不**強制升版；但建議升版以利追蹤。
- 消費端讀檔流程：
  1. 檢查 `version`；高於支援上限 → 警告但盡力解析（忽略未知欄位）。
  2. 以 `poseKeys` 補齊/排序 pose 軸；缺軸補預設（scale=1，其餘=0）。
  3. 修復 timeline：強制 `idle` 在首位、`frame` 遞增且唯一。
  4. 以 §3 範圍 clamp 作合法性檢查。

### 版本歷史
| version | 變更 |
|---------|------|
| 4 | 目前格式：**47 軸**，新增腕關節 `aL_wx`/`aL_wy`/`aR_wx`/`aR_wy` 與腳尖朝向 `lL_ty`/`lR_ty`；加大髖 Y（±150）、髖 Z（−60…120）範圍。遷移：v3→v4 缺的軸補預設 `0`，行為與舊版一致。|
| 3 | 41 軸，新增 `lL_contact` / `lR_contact`（腳掌接觸鎖，治墊腳/重心）。|
| 2 | 39 軸、有序 `seq`（含 `frame`/`returnFrames`）、`lags`、`dim`。|
| 1 | （PUNCH STUDIO 早期；舊「Godot text」與勾選覆寫格式，已不建議。）|

---

## 8. 最小範例

```jsonc
{
  "version": 4,
  "createdBy": "PUNCH STUDIO",
  "poseKeys": ["root_y","root_x","root_py","root_pz","sq","body_scale","squat",
    "spine_x","spine_y","pelvis_y","head_y","head_x","head_pz",
    "aL_sx","aL_sy","aL_sz","aL_ex","aL_idle","aL_scale",
    "aR_sx","aR_sy","aR_sz","aR_ex","aR_idle","aR_scale",
    "lL_hx","lL_hy","lL_hz","lL_kx","lL_ax","lL_idle","lL_scale",
    "lR_hx","lR_hy","lR_hz","lR_kx","lR_ax","lR_idle","lR_scale",
    "lL_contact","lR_contact",
    "aL_wx","aL_wy","aR_wx","aR_wy","lL_ty","lR_ty"],
  "seq": [
    { "name":"idle",   "frame":0,  "ease":"out", "impact":false, "tag":"idle",   "returnFrames":10 },
    { "name":"anti",   "frame":7,  "ease":"out", "impact":false, "tag":"anti"   },
    { "name":"strike", "frame":10, "ease":"in",  "impact":false, "tag":"strike" },
    { "name":"impact", "frame":14, "ease":"lin", "impact":true,  "tag":"impact" },
    { "name":"recovery","frame":26,"ease":"out", "impact":false, "tag":"recover"}
  ],
  "phases": {
    "idle":     { "squat":45, "aL_ex":40, "aR_ex":40, "lL_kx":40, "lR_kx":40 /* …其餘補預設 */ },
    "anti":     { "root_y":-18, "aR_sx":-60, "aR_ex":120 },
    "strike":   { "root_y":30,  "aR_sx":-100, "aR_ex":5 },
    "impact":   { "root_y":34,  "aR_sx":-108, "aR_ex":0 },
    "recovery": {}
  },
  "lags": { "aL":0.0, "aR":0.20, "lL":0.0, "lR":0.10 },
  "dim":  { "headSize":0.84, "bodyH":0.78, "bodyW":0.86, "bodyD":0.56,
            "armUpper":0.25, "armLower":0.30, "armThick":0.90, "armLenL":1.00, "armLenR":1.00,
            "legUpper":0.34, "legLower":0.45, "legThick":1.23, "fist":0.71, "shoe":1.11 }
}
```

> `phases` 內每組 pose 可只列「非預設」軸；消費端依 §3 補齊。
> 這份範例對應「後手直拳 cross」的精神，數值經裁剪示意。
