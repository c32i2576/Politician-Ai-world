# 外部事件注入模型（雙通道 + 稽核版）

> 版本：1 ｜ 更新日期：2026-04-30 ｜ 狀態：current
> 對應缺口：[../02-gaps.md §19](../02-gaps.md)
> 對齊：[time-model.md](time-model.md)（tick 為時間單位）、[media-model.md](media-model.md)（Event Stream 入口）
> 目的：把外部事件的注入通道、傳播路徑、稽核紀錄、PoC 最小實作釘死

---

## 1. 兩條通道，用途不同

外部事件注入有兩種性質，**不能合併**：

```
A. Admin Override（研究工具）
   → 直接改 world state，跳過媒體層
   → 用途：PoC 測試「把民調壓力調高 → 重跑」
   → 是測試工具，不是模擬世界內生的動作

B. World Event（情境事件）
   → 進 Event Stream，媒體 Agent 自決報不報、如何 frame
   → 用途：「注入股市崩盤事件，看議員如何反應」
   → 走正常流程，是模擬的一部分
```

合併兩條管道會讓「自然發展」與「研究者介入」混在同一條路徑，無法區分結果來自何處。

---

## 2. Schema

```typescript
externalEvents: defineTable({
  // 通道識別
  channel: v.union(
    v.literal("world_event"),    // 情境事件，進 Event Stream
    v.literal("admin_override"), // 直接改狀態，測試 / 研究用
  ),

  // 事件類型（channel 決定有效值，見 §3）
  eventType: v.string(),

  // 事件內容（structure 依 eventType 不同）
  payload: v.any(),

  // 觸發優先度（world_event 有效；admin_override 忽略此欄）
  priority: v.union(
    v.literal("normal"),
    v.literal("breaking"),    // 立即觸發 media tick，不等批次閘
  ),

  // 來源（用於稽核）
  source: v.union(
    v.literal("operator"),        // 研究者手動注入
    v.literal("scenario_fork"),   // 情境分岔觸發
    v.literal("news_feed"),       // 外部自動化 feed（未來擴充）
  ),

  // 影響範圍（world_event 用；admin_override 在 payload 內指定）
  targetDistrict: v.optional(v.string()),   // "taipei_1" | "national"
  targetTopic: v.optional(v.string()),

  // 時序（稽核核心）
  injectedAtTick: v.number(),               // 注入時的 tick
  processedAtTick: v.optional(v.number()),  // null = 尚未執行；有值 = 哪個 tick 處理
  // 注意：processedAtTick 有值後不刪除紀錄，稽核用
})
```

`processedAtTick` 用 `number | null` 取代 `boolean`，原因：
- 可以查「注入後幾個 tick 才生效」，若模擬卡住或跳 tick 會直接看出來
- 重現實驗時，可以驗証同一組注入序列在相同 tick 被處理

---

## 3. 事件類型清單

### 3.1 World Event（進 Event Stream）

| eventType | payload 結構 | priority 建議 | 對 shouldCloseSession 的影響 |
|-----------|-------------|--------------|------------------------------|
| `economic_shock` | `{ magnitude: number, direction: +1/-1 }` | normal | 無 |
| `natural_disaster` | `{ district: string, severity: 1-5 }` | breaking | 可觸發 crisis close |
| `international_event` | `{ topics: string[], direction: +1/-1 }` | normal | 無 |
| `external_scandal` | `{ targetAgentId?, description: string }` | breaking | 無（媒體自決） |
| `election_call` | `{ reason: string }` | breaking | 觸發 election close |

World Event 進入 Event Stream 後，由 [media-model.md §4](media-model.md) 的 `media_tick()` 決定是否報導、如何 framing。**研究者無法預知報導結果**，這是設計意圖——若需要確定性結果，改用 admin_override。

`external_scandal` 與 `LEAK_TO_MEDIA`（Agent 動作）的差異：前者不屬於任何 Agent，沒有 `source=leak` 的政治代價；後者帶 leaker 身分，媒體 framing 可能反咬 leaker。

### 3.2 Admin Override（直接改狀態）

| eventType | payload 結構 | 直接寫入目標 |
|-----------|-------------|-------------|
| `opinion_set` | `{ district, topic, sentiment: number, confidence?: number }` | `opinionState` |
| `stance_set` | `{ agentId, dimension, hiddenValue?, publicValue? }` | `stances` |
| `bill_advance` | `{ billId, toStage }` | `bills.stage` |
| `inject_memory` | `{ agentId, description, poignancy: 1-10, keywords: string[] }` | AssociativeMemory thought 節點 |

Admin Override **完全跳過媒體層**，直接寫 world state。`processedAtTick` 記下執行時機，確保實驗可重現。

---

## 4. Tick 執行位置

```
tick():
  1. loadWorld
  2. processExternalEvents()    ← 新增，優先於 Agent 動作
     ├─ world_event  → 推入 event_stream
     │                （priority=breaking → 立即觸發 media tick）
     └─ admin_override → 直接寫對應表
  3. handleAgentInputs()        ← 原有（Agent 動作佇列）
  4. tickAgents()               ← 原有
  5. saveWorld()                ← 原有
```

**為什麼在 step 2 而非 step 3？**

Agent 動作（step 3）由上一個 tick 的 LLM 決策產生，語意上是「本 tick 發生的事」。外部注入應該先於 Agent 動作生效，讓 Agent 在本 tick 的 `perceive()` 就能感知到——這樣才能模擬「看到新聞後立刻有所反應」。

---

## 5. 傳播路徑

### World Event 路徑（有媒體層）

```
operator 注入
  → externalEvents (channel: "world_event")
    → processExternalEvents() 推入 event_stream
      → media_tick() 評分 → 決定報導 / framing / spotlight
        → newsItems
          → apply_news_to_opinion()
            → Agent perceive() → plan() → execute()
```

### Admin Override 路徑（跳過媒體層）

```
operator 注入
  → externalEvents (channel: "admin_override")
    → processExternalEvents() 直接寫 opinionState / stances / bills
      → 下一個 tick Agent perceive() 時感知到狀態變化
         （但沒有對應的 newsItem，不會產生 media_spotlight）
```

注意：Admin Override 改了 `opinionState` 但不產生 `newsItems`，所以 `get_spotlight` 不會因此上升。若同時需要 spotlight 效果（例如模擬「輿論壓力 + 媒體曝光一起打」），需要分別注入一個 `admin_override(opinion_set)` 和一個 `world_event(external_scandal)`。

---

## 6. 與其他子系統的接口

| 子系統 | 接口點 |
|--------|--------|
| [media-model.md](media-model.md) | world_event 推入 event_stream；`priority=breaking` 觸發 `shouldRunMediaTick` 立即為 true |
| [time-model.md](time-model.md) | `election_call` / `natural_disaster(severity≥4)` 推入 `pendingEvents`，由 `shouldCloseSession` 判斷是否關閉 session |
| [bill-model.md](bill-model.md) | `bill_advance` 直接改 `bills.stage`；stage 變動進 event_stream（現有機制，不需額外處理）|
| [../01-research.md L394-L413](../01-research.md#L394-L413) 壓力函數 | `opinion_set` 改變 `opinionState.sentiment`，`constituency_pressure()` 在下一 tick 自動感知新值 |

---

## 7. PoC 最小實作

PoC 第 5 步「修改民調壓力 → 重跑」是 **restart 情境**，不需要即時注入：

| 步驟 | 內容 | 預估工時 |
|------|------|---------|
| 1 | `externalEvents` schema + `processedAtTick` 欄位 | 0.25 天 |
| 2 | `processExternalEvents()` 只實作 `admin_override` 分支 | 0.5 天 |
| 3 | Convex mutation `injectAdminOverride({ eventType, payload })` | 0.25 天 |
| 4 | PoC 測試：注入 `opinion_set` → 跑模擬 → 查 `processedAtTick` 確認生效 | 含在 PoC 工時內 |

**先砍掉**：
- `world_event` 通道（PoC 不需要情境事件注入）
- `scenario_fork` source（fork 機制是獨立子系統）
- `news_feed` source（外部 API 整合是未來功能）
- `bill_advance` / `inject_memory` admin override 類型（PoC 只需 `opinion_set`）

---

## 8. 主要 Tradeoff 與兜底

| Tradeoff | 解法 |
|---------|------|
| world_event 走媒體層 → 研究者無法預知效果 | 需要確定性結果改用 admin_override；兩條管道明確區分 |
| admin_override 直接寫狀態 → 沒有媒體 framing / spotlight | 有意為之；若需要一起打，分開注入兩條管道 |
| processedAtTick null 判斷比 boolean 多一步 | 稽核價值 > 程式便利性；查詢用 `.filter(e => e.processedAtTick === undefined)` |
| breaking priority 立即觸發 media tick → 可能打亂節奏 | `MEDIA_BATCH_SIZE` / `MAX_MEDIA_GAP` 雙閘不受影響，breaking 只是旁路插隊 |

---

## 9. 待決事項

- [ ] `world_event` 進 event_stream 時，`sourceType` 是 `"crisis"` 還是新增 `"external"`（影響 media_tick 的 `breaking_bonus` 計算）
- [ ] `election_call` 注入後是否需要 `MIN_TICKS_PER_SESSION` 的冷卻保護，還是強制立即 close session
- [ ] `stance_set` 的 admin_override：只改 `public` 還是 `hidden` 也可以改（研究用途允許直接設定 hidden 有助於建立對照組）
- [ ] 多個 admin_override 在同一 tick 注入同一欄位時（e.g., 兩次 opinion_set 同一 district/topic），取最後一個還是加性疊加
