# 媒體與輿論層（三通道 + 事件驅動版）

> 版本：2 ｜ 更新日期：2026-04-29 ｜ 狀態：current
> 對應缺口：[../02-gaps.md §3](../02-gaps.md)
> 取代／升級：[../archive/design-priority-three.md §3](../archive/design-priority-three.md)
> 對齊：[time-model.md](time-model.md)（事件驅動時鐘）
> 目的：把 `media_spotlight` 的源頭、`PUBLISH_POLL`／`RUN_AD_CAMPAIGN`／`LEAK_TO_MEDIA` 的角色、民調更新公式全部釘死

---

## 0. 與 v1 的差異

| 項目 | v1（design-priority-three §3） | v2（本檔） |
|------|------|------|
| 通道劃分 | 媒體 + 民調 兩層混用 | **三通道分離**：Media / Pollster / Campaign |
| `PUBLISH_POLL` 角色 | 含糊（「選民動作」） | **只揭露既有 sentiment + 採樣噪音**，不改變民意 |
| `LEAK_TO_MEDIA` 角色 | 直接產生新聞 | **僅進 event stream**，是否報導由 media agent 自決 |
| `RUN_AD_CAMPAIGN` 角色 | 未定義 | **付費通道**，繞過媒體直接寫 sentiment（含疲勞） |
| Media tick 頻率 | 每 round 跑 | **事件驅動 + 至少每 2 tick** |
| 民調更新 | `update_polls()` 黑盒 | **即時 apply_news + session close 定版** |
| 時間單位 | `time_window_days=3` | `time_window_ticks=3`（對齊 v2 時鐘）|
| Media Agent 心智 | 隱含三段式 | **規則式 score→top-K→framing**，不做 perceive/reflect |

PoC 建議直接走 v2，因為 v1 的 `update_polls()` 黑盒會讓「spotlight → 壓力 → 改票」的迴路無法驗證。

---

## 1. 三通道架構

```
┌─ Public Opinion State ─────────────────────────┐
│  per-(district, topic) sentiment + confidence  │  ← 唯一的「數值真相」
│  [-1, +1] 連續值，由三條通道獨立寫入           │
└────────────▲───────────────────────────────────┘
             │
   ┌─────────┼──────────┬──────────────────┐
   │         │          │                  │
┌──┴──────┐ ┌┴────────┐ ┌┴───────────────┐
│ Channel │ │ Channel │ │ Channel        │
│ A:Media │ │ B:Poll  │ │ C:Campaign     │
│         │ │         │ │                │
│ 改變    │ │ 揭露＋  │ │ 付費直寫       │
│ opinion │ │ 採樣噪音│ │ +疲勞          │
└──▲──────┘ └─────────┘ └────────────────┘
   │ 取材
┌──┴──────────────────────────────────────────┐
│ Event Stream                                 │
│  - 議員動作（vote / speech / press conf）    │
│  - LEAK_TO_MEDIA（觸發器，不保證見報）       │
│  - 突發事件（crisis / scandal）              │
└──────────────────────────────────────────────┘
```

### 1.1 三通道的角色釐清

| 通道 | 動作 | 對 `opinion.sentiment` 的影響 | 對 `opinion.confidence` 的影響 |
|------|------|------|------|
| A: Media | `PUBLISH_NEWS` / `FRAME_STORY` / `EDITORIAL_ENDORSE` | 加性更新（見 §3.1） | 高曝光 → 信心上升 |
| B: Pollster | `PUBLISH_POLL` | **不改變**，只快照 + noise | 揭露當下信心給其他 Agent |
| C: Campaign | `RUN_AD_CAMPAIGN` | 直接加，但受疲勞遞減（見 §3.2） | 不變 |

### 1.2 LEAK_TO_MEDIA 的定位

`LEAK_TO_MEDIA` **不是新聞**，它只是把一個事件塞進 event stream（標記 `source=leak`、可帶 `priority=breaking`）。是否被報導由 media agent 的 `score()` 決定。

這樣才能解釋：
- 「爆料被冷處理」（沒有 outlet 的 appetite/bias 接得住）
- 「同一爆料被不同媒體下不同框架」
- 「對手陣營主動爆料反而被反咬」（framing 被親 leaker 的 outlet 拿走）

---

## 2. Schema（升級 v1 §3.2）

```typescript
mediaOutlets: defineTable({
  name: v.string(),
  bias: v.record(v.string(), v.number()),   // per-topic bias [-1,+1]
  reach: v.number(),                         // 影響 spotlight 強度
  credibility: v.number(),                   // 0-1，影響對 sentiment 的傳遞係數
  appetite: v.union(
    v.literal("scandal"),
    v.literal("policy"),
    v.literal("horse_race")
  ),
})

newsItems: defineTable({
  outletId: v.id("mediaOutlets"),
  sourceEventId: v.string(),
  sourceType: v.union(                       // 新增：區分一般動作 vs leak
    v.literal("action"),
    v.literal("leak"),
    v.literal("crisis")
  ),
  subjectAgentId: v.optional(v.id("agents")),
  topic: v.string(),
  framing: v.union(
    v.literal("positive"),
    v.literal("neutral"),
    v.literal("negative"),
    v.literal("scandal")
  ),
  spotlightLevel: v.number(),                // 0-1
  publishedAtTick: v.number(),               // 改 tick（對齊 v2 時鐘）
})

opinionState: defineTable({                  // 新增：取代隱含的 polls 表
  district: v.string(),                       // 或 "national"
  topic: v.string(),
  sentiment: v.number(),                     // -1 ~ +1（真值，內部）
  confidence: v.number(),                    // 0-1
  lastUpdatedTick: v.number(),
  adFatigue: v.number(),                     // 累積廣告疲勞，per (district, topic)
})

polls: defineTable({                         // 改為「公開快照」表
  district: v.string(),
  topic: v.string(),
  reportedSentiment: v.number(),             // 真值 + sampling noise
  sampleSize: v.number(),
  publishedByAgentId: v.id("agents"),        // 誰發的（pollster / campaign / media）
  publishedAtTick: v.number(),
})
```

關鍵改動：**`polls` 從「真實民調」變成「公開的快照」**。Agent 看不到 `opinionState.sentiment` 的真值，只能看到 `polls` 表上的快照（含採樣噪音）。這樣 `PUBLISH_POLL` 才有戰略意義（誰發、何時發、樣本多大）。

---

## 3. 更新公式（補 v1 `update_polls()` 黑盒）

### 3.1 Media Channel：每則 newsItem 即時更新 sentiment

```python
def apply_news_to_opinion(news, district):
    s = opinion_state[district][news.topic]
    outlet = media_outlets[news.outletId]

    delta = (
        news.spotlightLevel
        * outlet.credibility
        * framing_sign(news.framing)            # +1 / 0 / -1 / -1.5(scandal)
        * district_receptivity(district, news.topic)
    )

    # 加性更新 + 向 baseline 回拉（避免單一媒體洗到極端）
    s.sentiment = clip(
        s.sentiment + ALPHA * delta - BETA * (s.sentiment - baseline[district][news.topic]),
        -1, 1
    )
    # 高曝光 → 信心上升（民眾覺得自己「知道得更多」）
    s.confidence = clip(s.confidence + GAMMA * news.spotlightLevel * outlet.credibility, 0, 1)
    s.lastUpdatedTick = current_tick
```

### 3.2 Campaign Channel：付費直寫，含疲勞

```python
def apply_ad(ad, district):
    s = opinion_state[district][ad.topic]
    saturation = exp(-FATIGUE_K * s.adFatigue)   # 疲勞越高，邊際效益越低
    s.sentiment = clip(
        s.sentiment + DELTA * ad.budget * saturation * ad.direction,
        -1, 1
    )
    s.adFatigue += ad.budget
    # 注意：不動 confidence（廣告不增加民眾「自覺知情度」）
```

疲勞每 session close 衰減：`adFatigue *= ADF_DECAY`（建議 0.6）。

### 3.3 Poll Channel：揭露 + 採樣噪音

```python
def publish_poll(pollster_agent, district, topic, sample_size):
    s = opinion_state[district][topic]
    noise = normal(0, 1 / sqrt(sample_size))
    poll_record = {
        reportedSentiment: clip(s.sentiment + noise, -1, 1),
        sampleSize: sample_size,
        publishedByAgentId: pollster_agent.id,
        publishedAtTick: current_tick,
    }
    insert(polls, poll_record)
    # 不改變 opinion_state.sentiment（這是觀測，不是介入）
```

但**民調本身會被媒體報導**，那則報導會走 §3.1 影響 sentiment——這是預期的二階效應，不是 bug。

### 3.4 預設係數（PoC 起跳值）

```typescript
const OPINION_CONFIG = {
  ALPHA: 0.15,          // news → sentiment 強度
  BETA: 0.05,           // 向 baseline 回拉
  GAMMA: 0.08,          // news → confidence
  DELTA: 0.0008,        // ad budget → sentiment（per 單位預算）
  FATIGUE_K: 0.5,       // 廣告疲勞曲率
  ADF_DECAY: 0.6,       // 每 session close 疲勞衰減
};
```

校準方式：PoC 後檢查「單則 scandal 對 sentiment 的衝擊」是否落在合理區間（建議 0.05-0.15），太大調低 ALPHA、太小調高。

---

## 4. Media Agent 決策（規則式，不做三段式心智）

```python
def media_tick(outlet, event_buffer):
    # 1. 評分每個事件
    scored = []
    for event in event_buffer:
        score = (
            appetite_match(event, outlet.appetite) * 0.35
          + scandal_factor(event) * 0.25
          + agent_prominence(event.actor) * 0.20
          + breaking_bonus(event) * 0.10              # leak/crisis 加成
          + random.uniform(0, 0.10)                    # 編輯室隨機性
        )
        scored.append((event, score))

    # 2. 取 top-K（K = ceil(outlet.reach * 5)）
    top_events = sorted(scored, key=lambda x: -x[1])[:K]

    # 3. 對每則決定 framing 與 spotlight
    for event, score in top_events:
        framing = decide_framing(event, outlet.bias)   # bias 主導，含小機率反向
        spotlight = clip(outlet.reach * score * framing_intensity(framing), 0, 1)
        news = emit_news_item(outlet, event, framing, spotlight)
        apply_news_to_opinion(news, district=event.district or "national")
```

**不做** perceive/plan/reflect 三段式。等 PoC 跑通後若需要「報社內鬥」敘事再升級。

---

## 5. Tick 頻率（對齊時鐘 v2）

```typescript
function shouldRunMediaTick(state: WorldState): boolean {
  return (
    state.eventBuffer.length >= MEDIA_BATCH_SIZE        // 攢到 N 件出刊
    || state.ticksSinceLastMediaTick >= MAX_MEDIA_GAP   // 至少每 N tick 一次
    || state.eventBuffer.some(e => e.priority === "breaking")
  );
}

const MEDIA_CONFIG = {
  MEDIA_BATCH_SIZE: 5,
  MAX_MEDIA_GAP: 2,
};
```

- 平時：事件積到 5 件或 2 tick 沒跑，就觸發一次 media_tick
- 突發：`breaking` 事件立即觸發（leak / crisis）
- Session close：強制 flush 一次 + 跑 `poll_aggregate`（將 `opinion_state` 快照寫入 polls 作為「定版民調」）

---

## 6. `media_spotlight` 的最終定義（修補壓力函數源頭）

```python
def get_spotlight(agent, topic, time_window_ticks=3):
    items = query_news(
        subjectAgentId=agent.id,
        topic=topic,
        since_tick=current_tick - time_window_ticks,
    )
    return sum(item.spotlightLevel * decay(current_tick - item.publishedAtTick) for item in items)
```

這個值直接餵給 [../01-research.md L401](../01-research.md#L401) 的 `pressures["media_spotlight"]`。

`decay(Δt) = exp(-0.4 * Δt)`，3 tick 後衰減到約 30%（PoC 校準）。

---

## 7. 民調對 Agent 壓力的傳導

```python
def constituency_pressure(agent, topic):
    poll = latest_poll(agent.district, topic)         # 看快照，不看真值
    if poll is None:
        return 0
    gap = poll.reportedSentiment - agent.public_stance[topic]
    urgency = 1 / max(ticks_to_election, 30)          # tick 為單位
    confidence_penalty = 1 / sqrt(poll.sampleSize)
    return gap * urgency * (1 - confidence_penalty)
```

注意 Agent 用的是 `polls.reportedSentiment`（含噪音）而非真值，這讓「樣本小的民調可被忽略」「對手放假民調」等敘事成立。

---

## 8. PoC 落地優先序

| 步驟 | 內容 | 預估工時 |
|------|------|---------|
| 1 | `opinionState` + `newsItems` schema + 2-3 家 outlet | 0.5 天 |
| 2 | `apply_news_to_opinion`（§3.1）+ `media_tick`（§4）規則式 | 1 天 |
| 3 | `get_spotlight`（§6）接到壓力函數 → 跑通改票迴路 | 0.5 天 |
| 4 | `PUBLISH_POLL` + `polls` 快照表 + `constituency_pressure` | 0.5 天 |
| 5 | session close 的 `poll_aggregate` | 0.5 天 |

**先砍掉**：
- `RUN_AD_CAMPAIGN`（不在 spotlight 關鍵路徑上，PoC 後再加）
- `FACT_CHECK` / `EDITORIAL_ENDORSE`（v1 §3.6 列了，但 PoC 不需要）
- 媒體 Agent 的 perceive/reflect 段（規則式即可）

---

## 9. 主要 Tradeoff 與兜底

| Tradeoff | 解法 |
|---------|------|
| 三通道分離 → schema 比 v1 複雜 | `opinionState` / `polls` / `newsItems` 表清楚，只多 1 張表 |
| Agent 看快照不看真值 → 多一層採樣模擬 | 採樣公式只有一行（normal noise），成本可忽略 |
| Media tick 頻率不固定 → 成本不可預測 | `MEDIA_BATCH_SIZE` + `MAX_MEDIA_GAP` 雙閘 |
| 規則式 media agent → 敘事深度不足 | PoC 通過後再升級為 LLM-driven framing |
| 廣告疲勞 K / 衰減 0.6 是猜的 | 進 `simulationConstants.ts`，PoC 後校準 |

---

## 10. 待決事項

- [ ] `district_receptivity` 是否要做 per-(district, topic) 的查找表，還是用 `bias_overlap` 即時算
- [ ] `framing_sign` 的 scandal 是 -1 還是 -1.5（scandal 是否比 negative 更重）
- [ ] 民調的 `sampleSize` 是 Agent 動作的參數，還是由 outlet/pollster 屬性決定
- [ ] `breaking_bonus` 是否應該也對非 leak 的 crisis 生效（地震、罷工等）
- [ ] cycle 邊界是否要對 `opinionState.confidence` 做 rebase（避免長期累積到 1.0）
