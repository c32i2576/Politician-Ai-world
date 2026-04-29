# 法案資料模型（Bill Schema）

> 版本：1 ｜ 更新日期：2026-04-29 ｜ 狀態：current
> 對應缺口：[../02-gaps.md §1](../02-gaps.md)
> 來源：[../archive/design-priority-three.md §1](../archive/design-priority-three.md) 抽出，並整合 [time-model.md §5.1](time-model.md) 的時序欄位修正
> 目的：把法案的條文結構、立場投影、流程狀態、版本鏈、時序釘死

---

## 1. 核心 Schema

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

  // 時序（對齊 time-model v2 — 綁 session id，不用 tick 數算期限）
  introducedAtTick: v.number(),              // 絕對 tick
  introducedAtSession: v.id("sessions"),
  expiresAtSession: v.optional(v.id("sessions")),  // 哪個 session 結束時失效
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
  tick: v.number(),
})
```

---

## 2. 關鍵設計決策

- **條文結構化**：不要存整段 markdown 法條文字。`articles[].tags` 是讓 LLM 與壓力函數知道「這條法案碰到哪些立場維度」的關鍵橋梁。
- **`stanceVector` 投影**：法案提出時就計算好（一次性 LLM 呼叫），之後 Agent 投票決策只要做向量點積就能得到「自然傾向」，再疊上壓力。避免每個 Agent 每次都要 LLM 重讀全文。
- **版本鏈**：修正案產生新版法案時 `parentBillId` 指向舊版，可追溯「妥協後變了什麼」。
- **`expiresAtSession` 屆期不續**：對應現實中的「會期不連續原則」，會期結束未三讀則退回。**直接綁 session id 而非 tick 數**——因為 v2 時間模型下 session 長度不固定（事件驅動），用 tick 數會算錯期限。
- **記名 / 不記名（`isRecorded`）**：直接影響 audience context，記名投票 `media_spotlight` 拉到最高，不記名讓 `hidden_stance` 浮現機率增加。

---

## 3. 與其他子系統的接口

| 子系統 | 接口 |
|--------|------|
| [time-model.md](time-model.md) | `expiresAtSession` 在 session close 時由 `bill_archive` 觸發處理 |
| [media-model.md](media-model.md) | bill stage 變動進 event stream，由 media agent 評分決定是否報導 |
| [../01-research.md L394-L413](../01-research.md#L394-L413) 壓力函數 | `stanceVector` × Agent stance 的內積 = 自然投票傾向，再疊上壓力 |

---

## 4. 待決事項

- [ ] `stanceVector` 維度是否要對齊 `STANCE_LAMBDA`（[time-model.md §3.2](time-model.md)）的 topic 分類（core_ideology / economic / tactical 等）
- [ ] 修正案 `billAmendments` 通過時是產生新 `bills` 列，還是直接改 `articles`？版本鏈策略待定
- [ ] 不記名投票時 `agentId` 是否仍記錄（用於後續分析）但對外 query 屏蔽
