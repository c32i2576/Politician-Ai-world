# 可觀測性模型

> 版本：1 ｜ 更新日期：2026-04-30 ｜ 狀態：current
> 對應缺口：[../02-gaps.md §5](../02-gaps.md)、[../02-gaps.md §7](../02-gaps.md)（成本監控接口）
> 對齊：[llm-cost-model.md](llm-cost-model.md)（SESSION_BUDGET_USD 讀取來源）、[time-model.md](time-model.md)（tick / session 邊界）
> 目的：定義三類可觀測數據的 schema 與計算方式，讓 PoC 驗收指標可以被量化查詢，並讓成本 budget cap 有數字來源

---

## 0. 觀測目標

可觀測性服務三個用途，**不互相混淆**：

| 用途 | 問題 | 對應資料 |
|------|------|---------|
| **工程健康** | 模擬有沒有卡住？成本有沒有爆？ | tick 時長、budget 累計、input queue 深度 |
| **行為驗收** | Agent 行為有沒有符合預期？ | 投票一致性、stance drift、backbone 效力 |
| **除錯追蹤** | 某個 Agent 為什麼改票？ | 單一 LLM 呼叫記錄、reflect 觸發鏈 |

三種用途對應三張表，查詢時不需要 JOIN 跨用途的資料。

---

## 1. Schema

### 1.1 llmCallLog（除錯追蹤 + 成本累計）

```typescript
llmCallLog: defineTable({
  // 呼叫身份
  agentId:    v.optional(v.id("agents")),  // null = 非 agent 呼叫（e.g., stanceVector 生成）
  callType:   v.string(),                  // "decideVote" | "negotiate" | "reflect" | "drafBill" | "plan"
  tier:       v.union(v.literal("A"), v.literal("B"), v.literal("C")),
  model:      v.string(),                  // "claude-haiku-4-5" | "claude-sonnet-4-6"

  // Token 用量
  inputTokens:  v.number(),
  outputTokens: v.number(),
  cachedTokens: v.number(),               // prompt cache 命中的 token 數

  // 成本（即時計算，方便直接 sum）
  costUsd:    v.number(),

  // 效能
  latencyMs:  v.number(),

  // 呼叫結果
  wasShortcut: v.boolean(),               // B 級：是否用點積捷徑跳過（costUsd = 0）
  success:     v.boolean(),               // false = LLM 回傳無效格式或逾時

  // 時序（用於 budget cap 查詢）
  tick:      v.number(),
  sessionId: v.id("sessions"),
}).index("by_session", ["sessionId"])     // budget cap 查詢：sum costUsd where sessionId = X
  .index("by_agent_tick", ["agentId", "tick"])  // 除錯：某 agent 在某 tick 的所有呼叫
```

**budget cap 的讀取方式**（接 [llm-cost-model.md §6](llm-cost-model.md)）：

```typescript
// 在每次 LLM 呼叫完成後更新累計
async function checkBudget(sessionId: Id<"sessions">): Promise<BudgetStatus> {
  const logs = await ctx.db
    .query("llmCallLog")
    .withIndex("by_session", q => q.eq("sessionId", sessionId))
    .collect();
  const totalCost = logs.reduce((sum, l) => sum + l.costUsd, 0);
  if (totalCost >= SESSION_BUDGET_USD * HARD_STOP_THRESHOLD)  return "hard_stop";
  if (totalCost >= SESSION_BUDGET_USD * DEGRADATION_THRESHOLD) return "degraded";
  return "normal";
}
```

---

### 1.2 tickSnapshot（工程健康）

```typescript
tickSnapshot: defineTable({
  tick:      v.number(),
  sessionId: v.id("sessions"),

  // 時間健康
  durationMs:        v.number(),   // 這個 tick 從開始到 saveWorld 的總時長
  llmWallTimeMs:     v.number(),   // tick 內所有 LLM 呼叫的累積等待時間

  // 佇列深度（在 tick 開始時快照）
  inputQueueDepth:   v.number(),   // 待處理的 Agent 動作數
  reflectQueueDepth: v.number(),   // 等待 reflect 的 Agent 數

  // 資源
  worldStateSizeKb:  v.number(),   // World 物件序列化後的大小
  sessionCostUsd:    v.number(),   // 本 session 截至本 tick 的累計費用（快取值，不重算）

  // 活躍度
  activeAgents:      v.number(),   // 本 tick 有 LLM 呼叫的 Agent 數
  shortcutRate:      v.number(),   // B 級呼叫中走捷徑的比例（0-1）
}).index("by_session", ["sessionId"])
```

---

### 1.3 stanceDriftLog（行為驗收）

每個 session close 時，對所有 Agent 的每個 stance 維度做一次快照：

```typescript
stanceDriftLog: defineTable({
  agentId:   v.id("agents"),
  sessionId: v.id("sessions"),
  tick:      v.number(),           // session close 的 tick 編號
  dimension: v.string(),           // "economic" | "environment" | "social" | ...

  hiddenValue:  v.number(),        // hidden_stance 當前值
  publicValue:  v.number(),        // public_stance 當前值
  drift:        v.number(),        // |hidden - public|，方便排序異常

  // PoC 驗收用：這個 session 內發生的 reflect 次數（追蹤 Agent 自我調整頻率）
  reflectCountThisSession: v.number(),
}).index("by_agent_session", ["agentId", "sessionId"])
  .index("by_drift", ["drift"])    // 快速找最大偏差 Agent
```

---

### 1.4 voteRecord 擴充（行為一致性分數的來源）

bill-model.md 的 `billVotes` 表已有 `vote` 欄位，補兩個觀測欄位：

```typescript
// 在 billVotes 的 schema 新增：
naturalLean:      v.number(),    // 投票當下的 dot_product(public_stance, bill.stanceVector)
pressureDelta:    v.number(),    // 投票結果與 naturalLean 的偏差方向（+1 / 0 / -1）
```

這兩個欄位在 `decideVote()` 執行時順手寫入，不需要額外查詢。

---

## 2. PoC 驗收指標（解決 #5）

有了上述 schema，原本模糊的兩個指標可以變成明確的查詢：

### 2.1 行為一致性分數

> 「預期立場 vs 實際投票」的一致率

```python
def behavior_consistency_score(session_id: str) -> float:
    votes = query_bill_votes(sessionId=session_id)
    
    consistent = 0
    total = 0
    for v in votes:
        if v.vote in ("yea", "nay"):       # 棄權 / 缺席不計
            expected = "yea" if v.naturalLean > 0 else "nay"
            if v.vote == expected:
                consistent += 1
            total += 1
    
    return consistent / total if total > 0 else None

# 通過門檻：PoC 目標 ≥ 0.70（7 成投票符合自然傾向）
# 失敗門檻：< 0.50（等於隨機）
```

**判定標準：**

| 分數 | 判定 | 意義 |
|------|------|------|
| ≥ 0.70 | ✅ 通過 | Agent 立場正確驅動投票行為 |
| 0.50–0.69 | ⚠️ 偏低 | 壓力函數過強或 prompt 不清 |
| < 0.50 | ❌ 失敗 | 立場與行為脫鉤，系統性問題 |

### 2.2 Stance Drift 分布（取代「符合直覺」）

> 「hidden 與 public 的偏差軌跡是否合理」

```python
def stance_drift_analysis(session_id: str) -> dict:
    logs = query_stance_drift_log(sessionId=session_id)
    
    return {
        # 最大偏差的 Agent（誰被壓力扭曲最多）
        "max_drift_agent": max(logs, key=lambda l: l.drift),
        
        # 高骨氣 vs 低骨氣 Agent 的平均偏差
        "high_backbone_avg_drift": mean([l.drift for l in logs if agent_backbone(l.agentId) >= 0.6]),
        "low_backbone_avg_drift":  mean([l.drift for l in logs if agent_backbone(l.agentId) < 0.4]),
        
        # 平均偏差
        "mean_drift": mean([l.drift for l in logs]),
    }

# 通過條件：high_backbone_avg_drift < low_backbone_avg_drift
# 含義：骨氣高的 Agent 立場被扭曲得比骨氣低的少 → backbone 機制有效
```

**為什麼這比「符合直覺」好：** 不需要人工判斷，只需要確認 high_backbone < low_backbone 這個方向關係。

---

## 3. 關鍵告警規則

寫入 `tickSnapshot` 後，在 tick 結束時檢查：

```typescript
function checkAlerts(snapshot: TickSnapshot): Alert[] {
  const alerts: Alert[] = [];
  
  if (snapshot.durationMs > 30_000)
    alerts.push({ level: "warn", msg: `Tick ${snapshot.tick} 超過 30 秒` });
  
  if (snapshot.worldStateSizeKb > 40)
    alerts.push({ level: "warn", msg: `World state 接近 50KB 上限` });
  
  if (snapshot.reflectQueueDepth > 10)
    alerts.push({ level: "warn", msg: `Reflect 佇列積壓 > 10，考慮調低 GLOBAL_CONCURRENT_CAP` });
  
  // 這個呼叫同時觸發 budget cap 邏輯（見 §1.1）
  if (snapshot.sessionCostUsd >= SESSION_BUDGET_USD * DEGRADATION_THRESHOLD)
    alerts.push({ level: "warn", msg: `費用達 ${snapshot.sessionCostUsd.toFixed(2)} USD，進入降級模式` });
  
  return alerts;
}
```

---

## 4. 與其他子系統的接口

| 子系統 | 接口點 |
|--------|--------|
| [llm-cost-model.md](llm-cost-model.md) | `llmCallLog.costUsd` 是 `SESSION_BUDGET_USD` 的讀取來源；budget 狀態由 `checkBudget()` 即時計算 |
| [bill-model.md](bill-model.md) | `billVotes` 表新增 `naturalLean` + `pressureDelta` 欄位；`behavior_consistency_score()` 讀這兩個欄位 |
| [time-model.md](time-model.md) | `stanceDriftLog` 在每個 session close 時寫入；`tickSnapshot` 每 tick 寫入 |
| [../01-research.md L394-L413](../01-research.md#L394-L413) 壓力函數 | `pressureDelta` 記錄壓力函數的實際效果，是校準 backbone / λ 的主要依據 |

---

## 5. PoC 最小實作

| 步驟 | 內容 | 預估工時 |
|------|------|---------|
| 1 | `llmCallLog` schema + 每次 LLM 呼叫後寫入（含 costUsd 即時計算） | 0.5 天 |
| 2 | `checkBudget()` 接入 tick 流程，讓 budget cap 有數字來源 | 0.25 天 |
| 3 | `billVotes` 補 `naturalLean` + `pressureDelta`，在 `decideVote()` 時寫入 | 0.25 天 |
| 4 | `stanceDriftLog` schema + session close 時批次寫入 | 0.25 天 |
| 5 | PoC 跑完後執行 `behavior_consistency_score()` + `stance_drift_analysis()` | 含在 PoC 工時內 |

**先砍掉：**
- `tickSnapshot`（PoC 規模不需要 tick 層健康監控，手動觀察即可）
- 告警規則（PoC 用 console log 代替）
- `reflectQueueDepth` 監控（PoC 只有 5 個 Agent，佇列不會積壓）

---

## 6. 主要 Tradeoff 與兜底

| Tradeoff | 解法 |
|---------|------|
| `llmCallLog` 累積快 → Convex storage 增加 | 每個 session close 後封存舊 log（搬到 `archivedLlmLogs`），保留最近 2 session |
| `stanceDriftLog` 只在 session close 快照 → 無法看 tick 內的細粒度變化 | PoC 驗收只需要 session 粒度；若需要 tick 粒度，改為每 N tick 快照一次 |
| `naturalLean` 在投票當下計算並儲存 → 增加每次投票的寫入量 | 兩個 number 欄位，寫入成本可忽略；比事後重算更可靠 |
| 成本 `costUsd` 即時計算 → 需要維護最新定價常數 | 定價放入 `simulationConstants.ts`，統一管理，與模型 ID 對應 |

---

## 7. 待決事項

- [ ] `behavior_consistency_score` 的通過門檻 0.70 是猜測值，需 PoC 後根據實際結果調整
- [ ] `stance_drift_analysis` 的「通過條件：high_backbone < low_backbone」是否要設數值門檻（e.g., 差距 > 0.1 才算顯著）
- [ ] `llmCallLog` 封存策略：session close 後立刻封存，還是保留 N session 的熱資料
- [ ] 是否需要一個 Convex `dashboard` query 把三張表的關鍵數字聚合成單一快照（方便前端顯示）
