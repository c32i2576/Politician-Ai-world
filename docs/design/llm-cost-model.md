# LLM 成本控制模型

> 版本：1 ｜ 更新日期：2026-04-30 ｜ 狀態：current
> 對應缺口：[../02-gaps.md §7](../02-gaps.md)、[../02-gaps.md §8](../02-gaps.md)
> 對齊：[time-model.md](time-model.md)（session/tick 邊界）、[politician-skill-contract.md](politician-skill-contract.md)（skillMd 大小）
> 目的：把 LLM 呼叫量、成本量級、Reflect 節流規則、降級策略釘死，讓「選 A/B/C/D 方案」有數字依據

---

## 1. LLM 呼叫分類（A / B / C 三級）

不是每個 Agent 每個 tick 都需要完整 LLM 呼叫。先把呼叫場景分三級：

| 級別 | 呼叫場景 | 執行條件 | 模型建議 |
|------|---------|---------|---------|
| **A 級**（永遠呼叫） | `negotiate()` 協商對話、`GIVE_SPEECH`、`FORM_COALITION` | 高語意複雜度，規則取代不了 | Sonnet |
| **B 級**（條件呼叫） | `decideVote()`、`drafBill()`、`plan()` | 看 §2 的捷徑判斷，可能跳過 | Haiku（捷徑失敗 → Sonnet） |
| **C 級**（可延遲） | `reflect()`、`perceive()` 摘要 | 看 §3 的節流規則 | Haiku |

**A 級永遠呼叫**的原因：協商和演講是模擬輸出的核心內容，用規則取代等於抽掉靈魂。

---

## 2. B 級呼叫：decideVote 捷徑

`decideVote()` 是最高頻的 A/B 邊界。**多數投票可以用點積捷徑完成，不需要 LLM**：

```python
def decide_vote(agent, bill) -> VoteDecision:
    # 自然傾向 = 立場向量 · 法案立場向量
    natural_lean = dot_product(agent.public_stance, bill.stanceVector)
    
    # 捷徑條件：傾向明確 + 黨紀方向一致 + 非記名投票
    if (
        abs(natural_lean) > VOTE_SHORTCUT_THRESHOLD   # 傾向夠強（建議 0.4）
        and party_whip_aligned(agent, bill)            # 黨紀方向一致
        and not bill.isRecorded                        # 不記名，媒體壓力低
    ):
        vote = "yea" if natural_lean > 0 else "nay"
        return VoteDecision(vote=vote, source="shortcut", rationaleNodeId=None)
    
    # 捷徑失敗 → 呼叫 LLM（完整推理）
    return llm_decide_vote(agent, bill)
```

**捷徑節省估算：** 在 PoC（最低工資法案 + 5 個立場分明的 Agent）中，預估 60-70% 的投票可以走捷徑。記名投票、修憲、黨紀衝突幾乎全走 LLM。

---

## 3. Reflect 節流機制（補 #8）

Reflect 的問題：政治高張力時每個 Agent 的 `importance_trigger_curr` 同時歸零，全部搶著 reflect。

### 3.1 三層節流

```python
class ReflectThrottle:
    PER_AGENT_COOLDOWN_TICKS = 3      # 每個 Agent 兩次 reflect 之間最少 3 tick
    GLOBAL_CONCURRENT_CAP   = 3      # 單一 tick 最多同時執行 3 個 reflect
    SESSION_REFLECT_RESERVE = True   # session close 的 reflect 不受 GLOBAL_CAP 限制
                                     # （session close 本來就是停機邊界）

def can_reflect(agent, global_state) -> bool:
    cooldown_ok = (current_tick - agent.lastReflectTick) >= PER_AGENT_COOLDOWN_TICKS
    slot_ok = global_state.reflectingCount < GLOBAL_CONCURRENT_CAP
    return cooldown_ok and slot_ok
```

### 3.2 優先佇列（slot 不夠時排隊）

```python
def get_reflect_candidates(agents, global_state) -> list[Agent]:
    eligible = [a for a in agents if can_reflect(a, global_state)]
    # importance_delta 最高的先 reflect
    sorted_by_urgency = sorted(eligible, key=lambda a: -a.importanceDelta)
    return sorted_by_urgency[:GLOBAL_CONCURRENT_CAP]
```

其餘進入 `reflectQueue`，下一個 tick 繼續排。**不丟棄，不跳過**，只是延遲。

### 3.3 Session Close Reflect 的特殊處理

Session close 時的 reflect 是**批次執行**，不受 `GLOBAL_CONCURRENT_CAP` 限制，但有獨立的批次預算上限（見 §5.2）。

---

## 4. Prompt Caching（最大單項成本節省）

每個 Agent 的 `skillMd` 約 18KB（~4,500 tokens），每次呼叫都要帶進 system prompt。

**不加 cache：**
100 agents × 10 呼叫/session = 1,000 次呼叫 × 4,500 tokens = 4.5M tokens（只是 skillMd 部分）

**加 cache（Anthropic prompt caching，TTL 5 分鐘）：**
- 同一 agent 的呼叫間隔通常 < 5 分鐘（同一 session 內）
- Cache hit rate 預估 80%
- 節省：4.5M × 80% = 3.6M tokens，cache read 費率是 write 的 10%
- 實際費用差：(3.6M × Sonnet write rate) vs (3.6M × Sonnet cache read rate)

**Cache 結構：**

```python
# System prompt 分成固定部分（可 cache）和動態部分（不 cache）
system_prompt = [
    {
        "type": "text",
        "text": FIXED_SYSTEM_HEADER,   # 模擬規則、角色定義
        "cache_control": {"type": "ephemeral"}
    },
    {
        "type": "text", 
        "text": agent.skillMd,          # 政治人物 profile（最大塊）
        "cache_control": {"type": "ephemeral"}
    },
    {
        "type": "text",
        "text": build_dynamic_context(agent, world_state)  # 不 cache，每次不同
    }
]
```

---

## 5. 成本估算（具體數字）

### 5.1 Token 用量假設

| 呼叫類型 | Input tokens | Output tokens | 說明 |
|---------|-------------|--------------|------|
| `decideVote`（LLM 路徑） | 6,000 | 500 | skillMd 4,500 + 世界狀態 1,500 |
| `negotiate` | 8,000 | 1,500 | skillMd + 對話歷史 |
| `drafBill` | 7,000 | 2,000 | skillMd + 法案草稿 |
| `reflect` | 5,000 | 800 | skillMd + 記憶摘要 |
| `plan`（session 開始） | 6,500 | 1,000 | skillMd + session context |

### 5.2 PoC 估算（5 agents、1 session、20 ticks）

| 項目 | 次數 | Input | Output | 費用（Haiku 4.5）|
|------|------|-------|--------|----------------|
| `decideVote`（LLM） | 5×3法案×40% = 6 | 36K | 3K | $0.03 |
| `decideVote`（捷徑） | 9 | — | — | $0 |
| `negotiate` | 3 | 24K | 4.5K | $0.02 |
| `reflect`（tick 中） | 4 | 20K | 3.2K | $0.02 |
| `reflect`（session close） | 5 | 25K | 4K | $0.02 |
| `plan`（session 開始） | 5 | 32.5K | 5K | $0.03 |
| **合計** | | **~137K** | **~20K** | **~$0.12** |

> Haiku 4.5 定價：$0.80 / 1M input、$4.00 / 1M output（截至本文）。  
> 加 prompt cache：skillMd 部分節省約 30%，實際 ~$0.09。

**PoC 成本極低，驗證期不是問題。**

### 5.3 生產規模估算（20 agents、1 session）

| 項目 | 次數 | Input | Output | 費用（Sonnet 4.6）|
|------|------|-------|--------|----------------|
| `decideVote`（LLM） | 20×5法案×40% = 40 | 240K | 20K | $0.72 + $0.30 |
| `negotiate` | 15 | 120K | 22.5K | $0.36 + $0.34 |
| `reflect`（tick 中） | 20 | 100K | 16K | $0.30 + $0.24 |
| `reflect`（session close） | 20 | 100K | 16K | $0.30 + $0.24 |
| `plan` | 20 | 130K | 20K | $0.39 + $0.30 |
| **合計（無 cache）** | | **~690K** | **~95K** | **~$3.19** |
| **合計（有 cache，估 70% hit）** | | — | — | **~$1.80** |

> Sonnet 4.6 定價：$3 / 1M input、$15 / 1M output、cache read $0.30 / 1M（截至本文）。

**1 個 session ≈ $1.80（有 cache）。1 個 cycle（16 sessions）≈ $29。可承受。**

### 5.4 100 agents 的邊界

100 agents 不是線性 × 5——因為每個 tick 只有少數 agents 有需要決策的事件。

估算每 tick 有 LLM 呼叫需求的 agents：~15%（15 人）。

| 規模 | 每 session 成本（有 cache） | 每 cycle 成本 |
|------|--------------------------|-------------|
| 5 agents | ~$0.09 | ~$1.44 |
| 20 agents | ~$1.80 | ~$29 |
| 50 agents | ~$4 | ~$64 |
| 100 agents | ~$7 | ~$112 |

**結論：100 agents + 1 個完整選舉週期 ≈ $112。研究用途可接受；若要跑大量平行情境分析需要更仔細的預算規劃。**

---

## 6. 降級策略（Budget Cap）

```typescript
const COST_CONFIG = {
  SESSION_BUDGET_USD:     5.00,   // 單一 session 的 token 費用上限
  DEGRADATION_THRESHOLD: 0.80,   // 達到 80% 預算時進入降級模式
  HARD_STOP_THRESHOLD:   0.95,   // 達到 95% 時暫停模擬
};
```

### 降級層級

```
正常模式（0%–80% 預算）
  → 所有 A/B/C 級呼叫正常執行

降級模式（80%–95% 預算）
  → C 級（reflect、perceive 摘要）全部延遲到 session close
  → B 級捷徑門檻從 0.4 → 0.25（更容易走捷徑，減少 LLM）
  → reflect 的 GLOBAL_CONCURRENT_CAP 從 3 → 1

緊急模式（> 95% 預算）
  → 暫停新的 LLM 呼叫
  → 正在執行的 A 級呼叫完成後停止 tick
  → 寫入 simulationStatus: "budget_paused"
  → 等人工確認後繼續
```

---

## 7. 常數集中管理

所有成本控制常數放入 `simulationConstants.ts`，與 `TIME_CONFIG`、`OPINION_CONFIG` 同層：

```typescript
const LLM_CONFIG = {
  // 呼叫分級
  VOTE_SHORTCUT_THRESHOLD:   0.4,   // B 級捷徑門檻
  DEGRADED_SHORTCUT_THRESHOLD: 0.25, // 降級模式捷徑門檻

  // Reflect 節流
  PER_AGENT_COOLDOWN_TICKS:  3,
  GLOBAL_CONCURRENT_CAP:     3,
  DEGRADED_CONCURRENT_CAP:   1,

  // 預算控制
  SESSION_BUDGET_USD:        5.00,
  DEGRADATION_THRESHOLD:     0.80,
  HARD_STOP_THRESHOLD:       0.95,

  // 模型路由
  MODEL_A_TIER: "claude-sonnet-4-6",   // A 級永遠 Sonnet
  MODEL_B_TIER: "claude-haiku-4-5-20251001",  // B 級優先 Haiku
  MODEL_C_TIER: "claude-haiku-4-5-20251001",  // C 級 Haiku
};
```

---

## 8. PoC 最小實作

| 步驟 | 內容 | 預估工時 |
|------|------|---------|
| 1 | `decideVote` 的點積捷徑（`VOTE_SHORTCUT_THRESHOLD = 0.4`） | 0.5 天 |
| 2 | Reflect per-agent cooldown（`lastReflectTick` 欄位 + 檢查）| 0.5 天 |
| 3 | `GLOBAL_CONCURRENT_CAP = 3` + 優先佇列 | 0.5 天 |
| 4 | Prompt cache 結構（固定/動態分層）| 0.5 天 |
| 5 | PoC 跑完後對照 §5.2 估算，驗證實際費用是否落在 $0.05–$0.20 | 含在 PoC 工時內 |

**先砍掉：**
- 降級模式（PoC 規模不會碰到預算上限）
- 模型路由（PoC 全用 Haiku 即可，Sonnet 留給生產）
- `SESSION_BUDGET_USD` hard cap（PoC 手動監控即可）

---

## 9. 主要 Tradeoff 與兜底

| Tradeoff | 解法 |
|---------|------|
| 捷徑誤差：點積捷徑可能在邊界案例判錯 | PoC 的「行為一致性分數」會抓到；調整 threshold 即可 |
| Reflect 延遲 → 感覺 Agent 反應遲鈍 | 延遲最多 PER_AGENT_COOLDOWN_TICKS tick；session close 必定跑 |
| 降級模式下 reflect 集中在 session close → session close 成本暴衝 | SESSION_BUDGET_USD 不計 session close reflect（視為固定成本）|
| Prompt cache TTL 5 分鐘 → tick 跑得慢時 cache 失效 | tick 不綁實時，在同一 session 內連續跑即可保持 cache warm |

---

## 10. 待決事項

- [ ] `SESSION_BUDGET_USD = 5.00` 是否合理，需 PoC 校準後調整
- [ ] 降級模式下，A 級的 `negotiate()` 是否也要加輪數上限（避免長協商燒光預算）
- [ ] `MODEL_A_TIER` 是否在 PoC 也用 Sonnet，還是全 Haiku 省錢（影響行為品質）
- [ ] Reflect 的 `importanceDelta` 計算方式（累積 poignancy delta？還是事件數？）
- [ ] 100 agents 情境下每 tick 活躍 agent 比例（15% 是估算，需實測）
