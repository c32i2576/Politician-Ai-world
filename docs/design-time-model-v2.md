# 時間／會期模型 v2（事件驅動版）

> 設計日期：2026-04-29
> 對應缺口：[architecture-gaps.md §2](architecture-gaps.md#L20-L26)
> 取代：[design-priority-three.md §2](design-priority-three.md#L98-L154) 的固定長度方案
> 目的：把「tick / 會期 / 選舉週期」三者對應釘死，並提供可重現的 stance 衰減公式

---

## 0. 與 v1 的差異

| 項目 | v1（design-priority-three §2） | v2（本檔） |
|------|------|------|
| tick 與現實時間綁定 | 1 tick = 30 秒實時 | **不綁現實時間**；1 tick = 1 邏輯回合 |
| 會期切換 | 固定 `DAYS_PER_PERIOD = 20` | **事件驅動**：agenda 清空或重大事件觸發 |
| 選舉週期 | `PERIODS_PER_TERM = 8` | 保留固定 K session = 1 cycle，cycle 邊界強制 reflect |
| stance 衰減 τ | 以「天」為單位 | 以 **tick** 為單位，確保可重現 |
| 衰減速率 | 全域單一 τ_stance | **per-topic λ**：核心價值慢、戰術立場快 |
| 效能上限 | 不適用（固定長度） | session hard cap 防失控 |

v1 的固定長度方案優點是成本可預測；v2 的優點是節奏更貼近真實議事，代價是需要 hard cap 兜底。**建議 PoC 用 v2**，若成本失控再降回 v1。

---

## 1. 三層時間軸

```
┌──────────────────────────────────────────────────────────┐
│  Cycle (選舉週期)                                          │
│  K sessions（建議 K=16，約 4 年）                          │
│  → 邊界強制觸發大型 reflect + stance rebase + 選舉        │
└──────────────────────────────────────────────────────────┘
                         ▲ 包含
┌──────────────────────────────────────────────────────────┐
│  Session (會期)                                            │
│  N ticks，N 不固定                                         │
│  事件驅動關閉：agenda 清空 / election / crisis / budget   │
│  Hard cap：N ≤ 20（防失控）                                │
│  → 邊界自動觸發 reflect                                    │
└──────────────────────────────────────────────────────────┘
                         ▲ 包含
┌──────────────────────────────────────────────────────────┐
│  Tick (邏輯回合)                                           │
│  1 tick ≈ 1 週敘事時間（in-world）                         │
│  不綁現實時間：UI 可暫停／快轉                             │
│  → 每 tick：perceive + plan + execute                     │
└──────────────────────────────────────────────────────────┘
```

### 為什麼 tick 不綁現實時間

- **可重現性**：給定 seed 和輸入序列，重跑結果一致（[architecture-gaps.md §17](architecture-gaps.md#L144)）。
- **UI 自由度**：使用者可暫停觀察、快轉看趨勢。
- **成本控制**：tick 速率由排程器決定，不被 wall clock 綁住。

「1 tick ≈ 1 週敘事時間」只是**敘事換算**，給 UI 顯示和 stance 衰減直覺用，不是引擎節拍。

---

## 2. Session 切換規則

### 2.1 觸發條件（任一成立即關閉 session）

```typescript
function shouldCloseSession(state: WorldState): boolean {
  return (
    state.agendaQueue.length === 0          // 議程清空
    || state.pendingEvents.some(e =>        // 重大事件
         e.type === "election"
      || e.type === "crisis"
      || e.type === "budget_vote"
      || e.type === "no_confidence"
    )
    || state.currentSession.tickCount >= MAX_TICKS_PER_SESSION  // hard cap
  );
}
```

### 2.2 常數

```typescript
const TIME_CONFIG = {
  // Tick 層
  TICKS_PER_NARRATIVE_WEEK: 1,      // 換算用，不影響引擎

  // Session 層
  MAX_TICKS_PER_SESSION: 20,        // hard cap
  MIN_TICKS_PER_SESSION: 3,         // 防止連續 close session 風暴

  // Cycle 層
  SESSIONS_PER_CYCLE: 16,           // 約 4 年
};
```

### 2.3 Session 邊界自動觸發

| 事件 | 時機 | 說明 |
|------|------|------|
| `reflect` (個體層) | session close 時 | 每個 Agent 對該 session 的事件做反思，產生新記憶 |
| `faction_check` | session close 時 | 派系凝聚度重算，可能分裂或合併 |
| `bill_archive` | session close 時 | 未進三讀的法案視觸發原因處理（清空→保留下期；crisis→強制歸檔） |
| `poll_aggregate` | session close 時 | 民調聚合定版，作為下期決策輸入 |

### 2.4 Cycle 邊界額外觸發

| 事件 | 說明 |
|------|------|
| `cycle_reflect` | Agent 對整個 cycle 做大反思，重評核心立場 |
| `stance_rebase` | hidden_stance 與 public_stance 重新對齊（避免長期漂移累積） |
| `election` | 席次重分配；落選 Agent 退場；新人加入 |
| `public_opinion_shift` | 結構性民意變化（generational shift） |

---

## 3. Stance 衰減公式

### 3.1 公式（補 [architecture-research.md L411](architecture-research.md#L411) 缺的部分）

```python
# 衰減速率以 tick 為單位（可重現）
new_stance = old_stance * exp(-λ_topic * Δtick) + evidence_pressure(Δtick)

# 其中：
# - old_stance：上一 tick 的公開立場 [-1, +1]
# - λ_topic：該議題的衰減係數（per-topic，見 §3.2）
# - Δtick：距離上次更新的 tick 數
# - evidence_pressure：本 tick 內收到的新證據／事件壓力
```

### 3.2 Per-topic λ 表

把 λ 綁議題類別，避免「核心價值跟戰術立場用同一速率」的失真：

```typescript
const STANCE_LAMBDA: Record<TopicCategory, number> = {
  // 核心價值：衰減慢，立場黏著
  core_ideology:        0.02,   // 半衰期 ≈ 35 ticks（約 8 個月敘事時間）
  constitutional:       0.03,   // 半衰期 ≈ 23 ticks
  identity:             0.025,  // 族群／國族議題

  // 政策立場：中等
  economic:             0.08,   // 半衰期 ≈ 9 ticks
  social_policy:        0.10,
  environment:          0.09,

  // 戰術立場：衰減快，容易隨壓力擺動
  tactical_alliance:    0.25,   // 半衰期 ≈ 3 ticks
  procedural_vote:      0.30,
  personnel:            0.35,
};
```

校準方式：PoC 跑完後，用「立場-投票一致性」反推 λ；若某議題 Agent 一直「翻來翻去」λ 太大，「立場僵死不動」λ 太小。

### 3.3 evidence_pressure 來源

```python
def evidence_pressure(agent, topic, tick):
    return (
        sum(news.spotlight * news.framing for news in news_this_tick(agent, topic))
        + sum(0.3 * msg.weight for msg in dm_received(agent, topic, tick))
        + 0.5 * faction_consensus_drift(agent.faction, topic, tick)
    )
```

具體權重在 PoC 校準。

---

## 4. Reflect 觸發頻率（取代「每 day 檢查」）

| 觸發時機 | 範圍 | 成本 |
|---------|------|------|
| Session close | 個體 reflect（該 session 的事件） | 中 |
| Cycle close | 個體 cycle reflect + stance rebase | 高 |
| 重大事件即時觸發 | 涉事 Agent 立即 reflect（不等 session close） | 低（單 Agent） |

不再有「每 N tick 強制 reflect」這種固定頻率，所有 reflect 都掛在語意邊界上。

---

## 5. 與其他子系統的接口

### 5.1 Bill 生命週期對應（修正 [design-priority-three.md L64-L65](design-priority-three.md#L64-L65)）

```typescript
bills: defineTable({
  // ...
  introducedAtTick: v.number(),       // 絕對 tick
  introducedAtSession: v.id("sessions"),
  expiresAtSession: v.optional(v.id("sessions")),  // 哪個 session 結束時失效
  // 不再用 expiresAt: number
})
```

理由：session 長度不固定，用 tick 數會算錯期限。直接綁 session id。

### 5.2 媒體層

`get_spotlight` 的 `time_window_days` 改為 `time_window_ticks`，預設 3 ticks（約 3 週敘事）。

### 5.3 集中常數

`TIME_CONFIG` + `STANCE_LAMBDA` 一併放入 `simulationConstants.ts`，與 `CLOCK_CONFIG`（v1 殘留）區分，方便 A/B 切換。

---

## 6. 主要 Tradeoff 與兜底

| Tradeoff | 解法 |
|---------|------|
| 事件驅動 → session 長度不可預測 → 成本不可預測 | `MAX_TICKS_PER_SESSION = 20` hard cap |
| 連續 close session 風暴（agenda 一直空） | `MIN_TICKS_PER_SESSION = 3` 強制延長 |
| Per-topic λ 增加調參維度 | 預設值起跳，PoC 後再用一致性分數反推 |
| Tick 不綁實時 → 使用者「無感」 | UI 顯示「敘事週」+ 進度條 |

---

## 7. 待決事項

- [ ] `MIN/MAX_TICKS_PER_SESSION` 預設值是否合理（需 PoC 驗證）
- [ ] `SESSIONS_PER_CYCLE = 16` 是否對應目標國家（台灣立院 vs 其他）的選舉週期
- [ ] λ 表的初始值是否要分黨派（保守派核心價值更黏？）
- [ ] cycle 邊界的 `stance_rebase` 演算法細節
