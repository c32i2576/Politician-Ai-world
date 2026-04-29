# 優先三項設計建議：法案 + 時間模型 + 媒體層

> ⚠️ **此檔已歸檔** — 內容已拆分並升級至：
> - §1 法案模型 → [../design/bill-model.md](../design/bill-model.md)
> - §2 時間模型 → [../design/time-model.md](../design/time-model.md)（v2，事件驅動）
> - §3 媒體層 → [../design/media-model.md](../design/media-model.md)（v2，三通道）
>
> 保留此檔作為歷史參考，**新工作請使用上述 current 版本**。

> 版本：1 ｜ 設計日期：2026-04-29 ｜ 狀態：archived
> 對應缺口：[../02-gaps.md](../02-gaps.md) 第 1、2、3 項
> 目的：在進入 PoC 實作前，把引擎跑得起來最關鍵的三個子系統釘下具體設計

---

## 1. 法案資料模型（Bill Schema）

### 1.1 核心 Schema

```typescript
bills: defineTable({
  // 識別
  billId: v.id("bills"),
  number: v.string(),              // "院總第1234號"
  title: v.string(),
  category: v.union(               // 影響委員會路徑
    v.literal("budget"),
    v.literal("amendment"),        // 修憲 → 觸發 SUPERMAJORITY
    v.literal("ordinary"),
    v.literal("treaty")
  ),

  // 內容（條文用結構化而非長文，方便 LLM 引用與修正）
  articles: v.array(v.object({
    articleNo: v.number(),         // 第 N 條
    text: v.string(),
    tags: v.array(v.string()),     // ["環保", "稅務"] → 對應立場維度
  })),

  // 立場投影：這部法案在各立場軸上的「方向」
  // 用來計算每個 Agent 的「自然投票傾向」
  stanceVector: v.object({
    economic: v.number(),          // -1 ~ +1
    environment: v.number(),
    social: v.number(),
  }),

  // 流程狀態
  stage: v.union(
    v.literal("drafted"),
    v.literal("first_reading"),
    v.literal("committee"),
    v.literal("second_reading"),
    v.literal("third_reading"),
    v.literal("passed"),
    v.literal("rejected"),
    v.literal("withdrawn")
  ),
  committee: v.optional(v.string()), // 目前所在委員會
  reviewRound: v.number(),           // 第幾輪委員會審查

  // 提案者與連署
  sponsorId: v.id("agents"),
  cosponsors: v.array(v.id("agents")),

  // 版本鏈（修正案家族）
  parentBillId: v.optional(v.id("bills")),  // 從哪個版本衍生
  version: v.number(),

  // 時序
  introducedAt: v.number(),          // session tick
  expiresAt: v.optional(v.number()), // 會期結束未過則失效
})

billAmendments: defineTable({
  billId: v.id("bills"),
  proposerId: v.id("agents"),
  targetArticleNo: v.number(),
  oldText: v.string(),
  newText: v.string(),
  status: v.union(v.literal("pending"), v.literal("adopted"), v.literal("rejected")),
})

billVotes: defineTable({
  billId: v.id("bills"),
  agentId: v.id("agents"),
  vote: v.union(v.literal("yea"), v.literal("nay"), v.literal("abstain"), v.literal("absent")),
  stage: v.string(),                 // 哪個階段的投票
  isRecorded: v.boolean(),           // 記名 vs 不記名 → 影響 audience
  rationaleNodeId: v.optional(v.string()), // 對應的 thought 節點
  timestamp: v.number(),
})
```

### 1.2 關鍵設計決策

- **條文結構化**：不要存整段 markdown 法條文字。`articles[].tags` 是讓 LLM 與壓力函數知道「這條法案碰到哪些立場維度」的關鍵橋梁。
- **`stanceVector` 投影**：法案提出時就計算好（一次性 LLM 呼叫），之後 Agent 投票決策只要做向量點積就能得到「自然傾向」，再疊上壓力。避免每個 Agent 每次都要 LLM 重讀全文。
- **版本鏈**：修正案產生新版法案時 `parentBillId` 指向舊版，可追溯「妥協後變了什麼」。
- **`expiresAt` 屆期不續**：對應現實中的「會期不連續原則」，會期結束未三讀則退回。
- **記名 / 不記名（`isRecorded`）**：直接影響 audience context，記名投票 `media_spotlight` 拉到最高，不記名讓 `hidden_stance` 浮現機率增加。

---

## 2. 時間／會期模型

> **本節已被 [../design/time-model.md](../design/time-model.md) 取代**（事件驅動 session + per-topic λ 衰減）。以下保留為歷史參考。


### 2.1 三層時鐘

```
Real-time tick (1 Hz)        ← 引擎執行單位
   ↓ 每 N ticks
Session round (約 5-10 min)  ← 一輪議事 = 多個 Agent 動作 + 一次 reflect 機會
   ↓ 每 M rounds
Session day                  ← 院會「一個議事日」
   ↓ 每 K days
Session period               ← 一個會期（法案大限）
   ↓ 每 P periods
Term                         ← 任期（影響選舉壓力、民調曲線）
```

### 2.2 建議參數

```typescript
const CLOCK_CONFIG = {
  TICKS_PER_ROUND: 30,        // 每輪 30 秒實時
  ROUNDS_PER_DAY: 8,          // 一個議事日 8 輪
  DAYS_PER_PERIOD: 20,        // 一個會期 20 個議事日
  PERIODS_PER_TERM: 8,        // 任期 8 個會期（約 4 年）
};
```

### 2.3 事件觸發對應表

| 觸發頻率 | 事件 |
|---------|------|
| 每 tick | `perceive` 視野更新；輸入佇列消化 |
| 每 round | Agent `plan/execute`（一個動作）；media tick |
| 每 day | 民調更新；reflect 重要性檢查；委員會審查推進一輪 |
| 每 period | 會期結算：未過法案歸檔；派系重整 |
| 每 term | 選舉：席次變動；民意 reset |

### 2.4 衰減公式（補回 ../01-research.md 缺的部分）

```python
# 記憶相關性評分
score = α·recency + β·relevance + γ·importance
# 其中：
recency = exp(-Δt_in_days / τ_recency)              # τ_recency = 30 天，半衰期約 21 天
importance_at_t = poignancy * exp(-Δt_in_days / τ_importance)  # τ_importance = 90 天

# stance 偏移衰減
public_stance_t+1 = hidden + (public_stance_t - hidden) * exp(-Δt / τ_stance)
# τ_stance = 14 days：壓力消失後兩週內公開立場會回歸真實立場
```

τ 值建議從這組起跳，再用 PoC 校準。

### 2.5 集中常數

把 `CLOCK_CONFIG` + `InstitutionalRules` + 各 τ 值集中在 `simulationConstants.ts`，方便日後做 rule-pack 切換（對應 [../02-gaps.md](../02-gaps.md) #16 在地化問題）。

---

## 3. 媒體與輿論層

> **本節已被 [../design/media-model.md](../design/media-model.md) 取代**（三通道分離、`apply_news_to_opinion` 公式、規則式 media_tick）。以下保留為歷史參考。

### 3.1 三層架構

```
┌──────────────────────────────────────┐
│  Public Opinion State (聚合層)        │
│  per-issue × per-district 民調指標   │
│  [-1, +1] 連續值 + 不確定度          │
└──────────────┬───────────────────────┘
               ↑ 受 News 影響
┌──────────────────────────────────────┐
│  Media Agents (媒體 Agent)            │
│  決定報導什麼、怎麼框架               │
└──────────────┬───────────────────────┘
               ↑ 從 events / leaks 取材
┌──────────────────────────────────────┐
│  Event Stream (議員動作 + 爆料)       │
└──────────────────────────────────────┘
```

### 3.2 Schema

```typescript
// 媒體 Agent
mediaOutlets: defineTable({
  name: v.string(),
  bias: v.object({                  // 媒體本身的立場偏向
    economic: v.number(),
    environment: v.number(),
  }),
  reach: v.number(),                // 觸及率：影響 spotlight 強度
  credibility: v.number(),          // 0-1，影響訊息對民調的傳遞係數
  appetite: v.union(                // 報導偏好類型
    v.literal("scandal"),
    v.literal("policy"),
    v.literal("horse_race")
  ),
})

// 新聞報導（每 round 由媒體 Agent 產生）
newsItems: defineTable({
  outletId: v.id("mediaOutlets"),
  sourceEventId: v.string(),        // 來自哪個議員動作或 leak
  subjectAgentId: v.optional(v.id("agents")),  // 主角是誰
  topic: v.string(),                // 對應立場維度
  framing: v.union(                 // 框架：影響輿論方向
    v.literal("positive"),
    v.literal("neutral"),
    v.literal("negative"),
    v.literal("scandal")
  ),
  spotlightLevel: v.number(),       // 0-1，給壓力函數用的關鍵欄位
  publishedAt: v.number(),
})

// 民調（聚合輿論指標）
polls: defineTable({
  district: v.string(),             // 選區或 "national"
  topic: v.string(),                // 議題維度
  sentiment: v.number(),            // -1 ~ +1
  confidence: v.number(),           // 樣本不確定度
  decayedFrom: v.number(),          // 上次更新時間
})
```

### 3.3 媒體 Agent 的決策邏輯（每 round）

```python
def media_tick(outlet, recent_events):
    # 1. 評分每個事件對該媒體的「報導價值」
    for event in recent_events:
        score = (
            match(event.topic, outlet.appetite) * 0.4 +
            scandal_factor(event) * 0.3 +
            agent_prominence(event.actor) * 0.2 +
            random.uniform(0, 0.1)             # 編輯室隨機性
        )

    # 2. 選 top-K 事件進行報導
    top_events = sorted(events, key=score)[:K]

    # 3. 對每個事件決定 framing（受 outlet.bias 影響）
    for event in top_events:
        framing = decide_framing(event, outlet.bias)
        spotlight = outlet.reach * score * framing_intensity(framing)
        emit_news_item(...)

    # 4. 更新民調（聚合所有 newsItems × credibility）
    update_polls()
```

### 3.4 `media_spotlight` 的計算（補回原文缺的源頭）

```python
def get_spotlight(agent, topic, time_window_days=3):
    # 從 newsItems 撈出近期該 agent 在該議題的曝光
    items = query_news(subjectAgentId=agent.id, topic=topic, since=now - 3d)
    return sum(item.spotlightLevel * decay(item.publishedAt) for item in items)
```

這個值就直接餵給 [../01-research.md L401](../01-research.md#L401) 的 `pressures["media_spotlight"]`。

### 3.5 民調對 Agent 壓力的傳導

```python
def constituency_pressure(agent, topic):
    poll = get_poll(agent.district, topic)
    gap = poll.sentiment - agent.public_stance[topic]
    urgency = 1 / max(days_to_election, 30)    # 選舉越近壓力越大
    return gap * urgency * (1 - poll.confidence_penalty)
```

### 3.6 媒體新動作集（補進 [../01-research.md L42-L46](../01-research.md#L42-L46)）

```
媒體動作：PUBLISH_NEWS, FRAME_STORY, FACT_CHECK, EDITORIAL_ENDORSE
議員-媒體：GIVE_INTERVIEW, REFUSE_COMMENT, OFF_THE_RECORD
```

---

## 4. 落地優先序

PoC 階段建議照這個順序刻：

| 步驟 | 內容 | 預估工時 |
|------|------|---------|
| 1 | Bill schema 最小版（articles + stanceVector + stage + votes） | 1 天 |
| 2 | Clock config + 三層時鐘 hook（先 round/day 兩層） | 0.5 天 |
| 3 | 單一 media outlet + spotlight 簡化版（用聚合公式而非完整 Agent） | 1 天 |
| 4 | 跑 [PoC 場景](../01-research.md#L508-L514)，驗證 spotlight → 壓力 → 改票 的迴路是否通 | 1-2 天 |

完整媒體 Agent、版本鏈、term-level 時鐘可以等 PoC 之後再補。

---

## 5. 與其他缺口的連動

| 本文設計項 | 同時解決 [../02-gaps.md](../02-gaps.md) 缺口 |
|-----------|--------------------------------------------|
| Bill schema | #1 法案資料模型 |
| 三層時鐘 + τ 公式 | #2 時間模型 |
| 媒體 Agent + 民調 | #3 媒體層、#5 PoC 指標的可量化基礎 |
| `simulationConstants.ts` 集中 | #16 在地化／規則可配置性 |
| `billVotes.isRecorded` | #13 資訊可見性的部分基礎 |

未涵蓋（仍待補）：#7 LLM 成本估算、#8 reflect 節流、#11 Politician-skill schema 銜接。
