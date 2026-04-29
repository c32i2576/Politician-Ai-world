# 政治模擬世界：架構研究

> 研究日期：2026-04-28  
> 參考專案：[MiroFish](../../../MiroFish)（社群輿論 AI 預測引擎）、[Politician-skill](../../../Politician-skill)、[ai-town](../../../ai-town)（多 Agent 即時模擬引擎）

---

## 專案目標

融合 MiroFish 的多 Agent 社群模擬架構與 Politician-skill 的 AI 政治人物蒸餾能力，
打造一個以「多 Agent 社交互動 + 知識圖譜記憶 + 決策/投票/立法」為核心的政治模擬世界。

---

## MiroFish 可複用的設計模式

| 模組 | MiroFish 做法 | 政治模擬借用方式 |
|------|--------------|-----------------|
| **本體生成** | LLM 分析文件 → 自動產生 10 種實體類型 | 改為生成議員/選民/媒體/黨團/委員會等政治實體 |
| **知識圖譜** | Zep Cloud 存關係網路與長期記憶 | 直接複用：存投票紀錄、聯盟關係、立場演變 |
| **Agent 設定** | LLM 生成個性 profile + 立場傾向 | Politician-skill 六軌研究輸出即為更精確的 profile 來源 |
| **平台動作** | Twitter/Reddit CRUD 動作集（CAMEL OASIS） | 替換為政治動作集（見下方） |
| **模擬執行** | CAMEL OASIS subprocess + JSONL 動作日誌 | 結構可複用，動作集與觸發規則須重寫 |
| **報告生成** | ReACT Agent + search/insight_forge/interview | 改為輸出法案通過率、派系消長、關鍵票分析 |
| **前端視覺化** | D3.js 互動圖譜 | 擴充為議場座位圖 + 派系力量圖 + 投票時間軸 |

**關鍵參考檔案（MiroFish）：**

- `backend/app/services/simulation_runner.py` — 模擬執行核心
- `backend/app/services/oasis_profile_generator.py` — Agent profile 生成
- `backend/app/services/report_agent.py` — ReACT 報告 Agent
- `backend/app/services/zep_tools.py` — 知識圖譜工具
- `docs/simulation.md` — 模擬流程文件

---

## 政治模擬：新增核心機制

### 1. 政治平台動作集（替換 Twitter/Reddit）

```
立法動作：INTRODUCE_BILL, AMEND_BILL, VOTE_YEA, VOTE_NAY, ABSTAIN, FILIBUSTER
社交動作：GIVE_SPEECH, HOLD_PRESS_CONFERENCE, LEAK_TO_MEDIA, FORM_COALITION
黨團動作：ISSUE_PARTY_WHIP, BREAK_WITH_PARTY, NEGOTIATE_DEAL
選民動作：RALLY_SUPPORT, PUBLISH_POLL, RUN_AD_CAMPAIGN
```

### 2. 制度規則引擎（MiroFish 無此模組，需新增）

```python
# institutional_rules.py
class InstitutionalRules:
    QUORUM_THRESHOLD = 0.5       # 過半出席才能投票
    SUPERMAJORITY = 2/3          # 修憲門檻
    COMMITTEE_REVIEW_ROUNDS = 2  # 委員會審查輪數
    FILIBUSTER_BREAK = 60        # 終結辯論票數
```

### 3. Agent 記憶層擴充（建在 Zep 之上）

現有 Zep 記憶，加入：
- 投票紀錄時間線
- 承諾／食言追蹤
- 與其他議員的信任分數
- 選區民調壓力
- 黨紀約束強度

### 4. 動態派系形成

- 每輪結算各 Agent 的立場向量（經濟左右 × 社會保守進步）
- 立場相近者自動形成臨時聯盟
- 聯盟記入知識圖譜，影響後續投票行為

---

## 整體架構分層

```
┌─────────────────────────────────────────────────────────┐
│  Politician-skill  (輸入層)                              │
│  六軌研究 → JSON profile (立場向量/語言/策略/選區)        │
└────────────────────────┬────────────────────────────────┘
                         ↓ 初始化 Agent profile
┌─────────────────────────────────────────────────────────┐
│  Game Engine  (ai-town convex/engine 模式)               │
│  ┌──────────────┐  ┌─────────────────┐  ┌────────────┐  │
│  │  World State │  │  inputs table   │  │  tick()    │  │
│  │  < 幾十 KB   │  │  動作佇列        │  │  1 Hz step │  │
│  │  議員+法案   │  │  VOTE/BILL/...  │  │  single-   │  │
│  │  +聯盟狀態  │  │  single-thread  │  │  threaded  │  │
│  └──────────────┘  └─────────────────┘  └────────────┘  │
│  InstitutionalRules: 法定人數 / 修憲門檻 / 委員會輪數    │
└────────────────────────┬────────────────────────────────┘
          ↓ startOperation              ↓ archive tables
┌─────────────────────┐   ┌──────────────────────────────┐
│  Agent LLM Ops      │   │  Memory Layer (雙層)          │
│  (internalAction)   │   │  個人記憶：Convex vector DB   │
│  decideVote()       │   │  (ai-town 模式，對話摘要      │
│  drafBill()         │   │   + embedding)               │
│  negotiate()        │   │  關係圖：Zep Cloud            │
│  → 呼叫 Claude API  │   │  (派系/聯盟/承諾追蹤)         │
└─────────────────────┘   └──────────────────────────────┘
                         ↓
┌─────────────────────────────────────────────────────────┐
│  Report Agent (ReACT)                                    │
│  法案通過率 + 派系消長 + 關鍵票分析 + 食言追蹤           │
└────────────────────────┬────────────────────────────────┘
                         ↓
┌─────────────────────────────────────────────────────────┐
│  Frontend  (React + Convex useQuery hooks)               │
│  議場座位圖 (PixiJS) + 派系力量圖 (D3) + 投票時間軸      │
└─────────────────────────────────────────────────────────┘
```

**資料流說明：**
- 引擎 tick 觸發 `startOperation` → LLM 非同步決策 → 結果寫回 `inputs` → 下一個 tick 生效
- 前端只用 Convex `useQuery` 訂閱，無需 WebSocket 手動管理
- 歷史法案/對話結束後封存至 `archivedBills` / `archivedConversations`，不佔用熱路徑

---

## 與 MiroFish 的差異點（需從頭設計）

| 差異點 | 說明 |
|--------|------|
| **制度規則引擎** | MiroFish 無議事規則；政治模擬需要硬性規則（法定人數、修正程序、委員會） |
| **因果追蹤** | 需記錄「誰因為什麼原因改票」，社群模擬只需記錄行為結果 |
| **雙層 Agent** | 政治人物（Politician-skill 精確 profile）+ 一般選民（程序性生成），兩層分開處理 |
| **勝負判定** | 社群模擬沒有「通過/否決」二元結果；立法模擬需要明確的終局判定 |
| **時間尺度** | 社群輿論以「小時」為單位；立法週期以「會期/天」為單位 |

---

---

## Generative Agents（Stanford Reverie）可借用的設計模式

> 原作：Joon Sung Park et al. (2023)「Generative Agents: Interactive Simulacra of Human Behavior」  
> 程式碼：`reverie/backend_server/`（Python）

### 核心認知迴圈

Generative Agents 最重要的貢獻是這個五步驟認知序列，每個 Agent 每個時間步都會執行：

```
perceive(maze)
  → 感知視野內的事件（按 att_bandwidth 過濾，retention 去重）
    ↓
retrieve(perceived)
  → 從記憶庫取回相關事件與想法
  → 評分公式：score = α·recency + β·relevance(cosine) + γ·importance(poignancy)
    ↓
plan(maze, personas, new_day, retrieved)
  → 長期：每天重新生成 daily_req（今天要做什麼）
  → 短期：將 daily_req 分解為小時級行程
  → 反應性：若有意外事件（人出現、對話邀請），插入即時計畫
    ↓
reflect()
  → 條件觸發：accumulated_importance > importance_trigger_max
  → 步驟：抽取 focal_points → 檢索相關記憶 → LLM 生成洞見（insight）→ 寫入新 thought 節點
    ↓
execute(maze, personas, plan)
  → 輸出：(next_tile, pronunciatio_emoji, description)
```

**政治模擬對應：**  
`perceive` → 感知議場事件（法案提出、盟友叛黨、媒體爆料）  
`plan` → 每「會期」重新生成立法議程；每「輪」分解為具體動作  
`reflect` → 累積足夠政治事件後生成策略洞見（「我應該轉換立場」）  
`execute` → 輸出政治動作（VOTE_YEA、FORM_COALITION、LEAK_TO_MEDIA）

---

### 三層記憶結構（最可複用的設計）

```
┌─────────────────────────────────────────────────────┐
│  Scratch（短期工作記憶）                              │
│  curr_time, curr_tile, daily_req, 當前行動           │
│  身份特質：innate(L0) / learned(L1) / currently(L2) │
│  超參數：vision_r=4, att_bandwidth=3, retention=5   │
└─────────────────────────────────────────────────────┘
┌─────────────────────────────────────────────────────┐
│  AssociativeMemory（記憶流 / Memory Stream）          │
│  節點類型：event / thought / chat                    │
│  每個 ConceptNode 包含：                             │
│    subject-predicate-object 三元組（SPO）            │
│    description, keywords, poignancy(1-10)           │
│    embedding（向量），last_accessed                  │
│    expiration（可設定遺忘時間）                       │
└─────────────────────────────────────────────────────┘
┌─────────────────────────────────────────────────────┐
│  SpatialMemory（空間記憶樹）                          │
│  world > sector > arena > [objects]                 │
│  Agent 知道哪些地方存在什麼物件                      │
└─────────────────────────────────────────────────────┘
```

**政治模擬的記憶改造：**

| 原始 | 政治版本 |
|------|---------|
| SPO: `(Isabella, is, at_cafe)` | `(王委員, voted_yea, 最低工資法案)` |
| SPO: `(Klaus, is_talking_with, Maria)` | `(李黨鞭, issued_whip_against, 環保條款)` |
| poignancy 1-10 | 重要性：破壞黨紀=9，例行表決=2 |
| SpatialMemory: 咖啡廳→廚房→桌子 | 制度地圖：院會→委員會→黨團室→媒體室 |
| daily_req: 今天要畫畫 | 會期議程：推進法案X、阻止法案Y、拉攏Z |

---

### Reflect 模組：政治模擬最有價值的功能

Reflect 的觸發條件是「累積重要性超過閾值」，而非固定頻率。這在政治模擬中特別有用：

```python
# 原版邏輯（可直接借用）
if persona.scratch.importance_trigger_curr <= 0:
    # 達到閾值 → 觸發反思
    focal_points = generate_focal_points(persona, n=3)
    # e.g., ["王委員最近的投票模式", "黨紀壓力", "選區民調下滑"]
    
    for focal_pt in focal_points:
        retrieved_nodes = retrieve_by_focal(focal_pt)
        insights = generate_insights_and_evidence(persona, retrieved_nodes)
        # e.g., {"我應該在環保議題上採取更積極立場": [node_12, node_15, node_18]}
        
        # 洞見寫入 AssociativeMemory 成為新的 thought 節點
        # → 影響後續 plan() 的決策
```

這個機制讓 Agent **從自己的行為歷史中學習並調整策略**，正是政治模擬需要的「立場演變」功能。

---

### Fork 機制：情境分岔（政治模擬獨特需求）

Reverie 的每個模擬都 fork 自前一個：

```python
# 政治模擬應用：情境分析
sim_A = ReverieServer("base_parliament_session1", "scenario_A_tax_bill_pass")
sim_B = ReverieServer("base_parliament_session1", "scenario_B_tax_bill_fail")
# 從同一起點分叉，比較兩種結果下的長期政治影響
```

**用途：**
- 「如果王委員那一票投了反對票，後續三輪會怎麼發展？」
- 媒體爆料事件的影響分析（在爆料前後各 fork 一條時間線）

---

### 與 ai-town 的技術對比（決策參考）

| 維度 | Generative Agents（Reverie） | ai-town |
|------|------------------------------|---------|
| **語言** | Python | TypeScript (全端) |
| **後端** | Django + 檔案系統通訊 | Convex（即時 DB） |
| **前端同步** | 檔案 JSON 輪詢（低效） | Convex 訂閱（即時推送）|
| **認知深度** | 完整五步驟迴圈 + Reflect | tick + startOperation（較輕量）|
| **記憶系統** | 三層記憶（完整，含 poignancy） | 向量 DB（較簡單）|
| **Fork/分岔** | 原生支援 | 無 |
| **並發能力** | 循序執行（每 Agent 依序） | single-threaded per world |
| **擴充難度** | 改 Python 模組，架構清晰 | 改 TypeScript，需懂 Convex |

**建議：認知層用 Reverie 模式，即時通訊層用 Convex**  
→ 將 Reverie 的 `perceive→retrieve→plan→reflect→execute` 實作為 Convex `internalAction`，  
　　利用 Convex 處理狀態同步與前端推送，Reverie 的認知邏輯作為 LLM 呼叫的 prompt 設計藍圖。

---

## ai-town 可借用的技術模式

ai-town 是以 **Convex** 為後端的即時多 Agent 模擬引擎，架構比 CAMEL OASIS 更乾淨、TypeScript-first，適合做為替代或補充選項。

### 核心架構對照

| ai-town 概念 | 政治模擬對應 | 技術細節 |
|-------------|-------------|---------|
| `World` | 議會/選區世界 | 單一文件存所有 Player + Agent + Conversation，保持 < 幾十 KB |
| `Player` | 議員/選民角色 | 位置 → 換成「立場向量」(經濟軸 × 社會軸)；移動 → 換成「立場漂移」 |
| `Conversation` (2人) | 院內協商/遊說 | 擴充為多人：委員會審查、全院辯論 |
| `Agent.tick()` | 每輪 Agent 行為決策 | tick 觸發動作：決定是否提案、遊說、公開發言 |
| `inputs` table | 政治動作輸入佇列 | `VOTE_YEA`、`INTRODUCE_BILL` 等動作走同一管道，保持 single-threaded |
| `memories` (向量 DB) | Agent 長期記憶 | 對話後 LLM 摘要 → embedding → Convex vector DB；換成「政治事件摘要」 |
| `participatedTogether` 圖 | 互動關係圖 | 直接複用：記錄「哪兩位議員曾協商/衝突」 |
| `archivedConversations` | 歷史立法紀錄 | 歷屆法案結果封存，供記憶層查詢 |

### ai-town 引擎核心機制（可直接複用）

```
Step 執行流程（每秒 1 次）：
  1. loadWorld  — 從 DB 載入整個世界狀態到記憶體
  2. handleInput — 依序處理輸入佇列（投票/提案/協商動作）
  3. tick(now)  — 推進時間：立場漂移、聯盟結算、任期倒數
  4. saveWorld  — diff 寫回 DB；刪除結束的 Conversation 並封存
```

**關鍵設計保證：** 每個 World 的引擎是 single-threaded（generation number 防止並發），  
完全不需要考慮 race condition — 投票計數、法定人數判斷可直接在 tick 內做。

### 記憶層：ai-town vs Zep Cloud

| | ai-town 內建 | Zep Cloud（MiroFish 用） |
|---|---|---|
| **儲存** | Convex vector DB（embedding） | 外部服務，關係圖 + 向量 |
| **記憶生成** | 對話結束後 LLM 摘要 → embedding | 自動提取實體/關係 |
| **查詢** | "What do you think about X?" → top-K 相似記憶 | 圖遍歷 + 語意搜尋 |
| **政治模擬適合度** | 適合個人記憶（「我上次和王委員談到稅改」） | 適合關係網路（「派系聯盟、勢力消長」） |
| **建議** | **兩層並用**：個人記憶用 ai-town 模式；關係圖用 Zep | — |

### 重要限制（ai-town 引擎邊界）

- 遊戲狀態必須保持 **< 幾十 KB**（全部載入記憶體）  
  → 議員人數建議 ≤ 100 人；歷史投票紀錄移入 archive tables
- 不適合高頻輸入（每秒數千次）→ 立法模擬不受影響（動作頻率低）
- 不適合計算密集型 tick → LLM 呼叫需移出引擎，用 `startOperation` 非同步執行

### ai-town 的 Agent 非同步 LLM 呼叫模式

```typescript
// Agent.tick() 內 — 只做狀態判斷，不直接呼叫 LLM
if (shouldDecideVote(agent, gameState)) {
  agent.startOperation(ctx, internal.agent.politicianOps.decideVote, {
    agentId: agent.id,
    billId: currentBill.id,
  });
}

// decideVote (internalAction) — 非同步，可呼叫 LLM
// 1. 載入 Agent 的記憶（過去投票、選區壓力、黨紀）
// 2. 呼叫 Politician-skill 生成的 profile prompt
// 3. LLM 回傳投票決定
// 4. ctx.runMutation(insertInput, { type: 'VOTE_YEA', billId, agentId })
```

---

## 實作路徑選項

### 選項 A：Fork MiroFish，替換動作集（快，3-4 週）
- 複製 MiroFish backend，替換 `simulation_runner.py` 動作集
- 新增 `institutional_rules.py`
- 前端加入投票面板
- **優點：** 快速驗證  
- **缺點：** 綁死 CAMEL OASIS 版本

### 選項 B：以 MiroFish 為藍圖，從零建新 repo（穩，6-8 週）
- 重新設計 Agent 動作協議（不依賴 CAMEL OASIS）
- 自行實作制度規則引擎
- 與 Politician-skill 深度整合
- **優點：** 架構乾淨，長期可擴充  
- **缺點：** 工程量大

### 選項 C：先做 PoC 驗證核心假設（推薦起點，1-2 週）
- 用 3-5 個 Politician-skill 生成的 Agent
- 手刻一個簡單的投票場景（無框架依賴）
- 驗證「知識圖譜記憶能否影響 Agent 投票行為」這個核心假設
- 再根據結果決定 A 或 B

### 選項 D：Fork ai-town，替換 World 語意（推薦長期方案）
- Fork ai-town repo，保留 `convex/engine` 完整引擎
- 替換 `convex/aiTown/` → `convex/politicalWorld/`：Player→Politician、Conversation→Negotiation
- 新增 `convex/politicalWorld/bill.ts`（法案)、`institutionalRules.ts`（制度規則）
- 前端：Pixi.js 議場視覺化替換 tile map；保留 Convex `useQuery` hooks 架構
- **優點：** 引擎穩定可靠（generation number、single-threaded、diff save 均現成）；TypeScript 全端型別安全
- **缺點：** 需要學習 Convex；無法直接複用 MiroFish Python 後端

---

## Agent 多面性模型：立場漂移與合理化機制

政治人物在群眾、利益、黨紀面前會修改立場。這需要一個獨立的子系統來模擬。

### 雙層立場架構

每個 Agent 同時維護兩個立場向量，彼此可以不同：

```python
class PoliticianStance:
    # 私有立場（hidden）：Agent 真正相信的，不直接暴露
    hidden:  { "economic": -0.3, "environment": +0.7, "tax_reform": -0.5 }
    
    # 公開立場（public）：對外宣稱的，受壓力影響後偏移
    public:  { "economic": +0.1, "environment": +0.6, "tax_reform": +0.2 }
    
    # 骨氣分數（來自 Politician-skill profile，固定特質）
    backbone: 0.3   # 0=完全順從, 1=鐵板一塊
```

`public` 是 `hidden` 被壓力函數扭曲後的結果。`backbone` 決定扭曲的幅度上限。

---

### 壓力函數（每輪計算公開立場偏移量）

```python
def compute_stance_shift(agent, context) -> StanceVector:
    pressures = {
        "party_whip":       agent.party.whip_strength * whip_direction,
        "constituency_poll": poll_gap * urgency,        # 民調與立場落差
        "donor_interest":   donor_pressure * direction, # 金主施壓方向
        "media_spotlight":  spotlight * risk_aversion,  # 曝光度越高越保守
        "coalition_need":   coalition_ask * trust_score # 盟友要求
    }
    
    total_pressure = sum(pressures.values())
    
    # backbone 作為抵抗係數：backbone 越高，偏移越小
    shift = total_pressure * (1 - agent.backbone) * time_decay
    
    agent.public_stance = agent.hidden_stance + shift
    return shift
```

---

### 觀眾切換（Audience Context）

同一個 Agent 在不同場合呈現不同面貌。每次執行動作時，LLM prompt 注入當前觀眾情境：

```python
audience_context = {
    "party_caucus":    { "observers": ["黨鞭", "派系大老"], "stakes": "high"   },
    "public_hearing":  { "observers": ["媒體", "公民"],     "stakes": "medium" },
    "private_negotiation": { "observers": ["單一議員"],     "stakes": "low"    },
    "floor_vote":      { "observers": ["全院", "直播鏡頭"], "stakes": "highest"}
}

# 在 execute() 的 prompt 中注入：
# "現在你在 {audience}，{observers} 正在看著你，
#  你的公開立場是 {public_stance}，你真實的想法是 {hidden_stance}，
#  你會如何行動？"
```

**關鍵規則：**
- 記名投票（`floor_vote`）：`media_spotlight` 最高，`public_stance` 主導
- 不記名或程序性投票：`hidden_stance` 浮現機率增加
- 黨團密室（`party_caucus`）：可表達真實反對，但未必翻臉

---

### 合理化敘事（Rationalization via Reflect）

當 `|public_stance - hidden_stance|` 超過閾值，觸發 Reflect 生成自我說服：

```
條件：立場偏移量 > rationalization_threshold

focal_points（自動生成）：
  → "我剛剛投票支持了我原本反對的條款"

retrieved_memories（相關記憶）：
  → [黨鞭施壓事件, node_12]
  → [選區民調下滑, node_15]  
  → [委員會席次威脅, node_18]

LLM 生成洞見（寫入 thought 節點）：
  → "為了保住財政委員會席次，這次在稅改上的妥協是必要的"
  → "我可以在下一個法案上補回來"

效果：
  → 這個 thought 節點成為未來決策的自我說服材料
  → Agent 不只是「被推著走」，而是主動建構敘事
  → 記憶鏈完整保存，可事後追蹤「誰因何改票」
```

---

### 立場演變的觸發事件清單

| 事件類型 | 壓力強度 | 主要作用 |
|---------|---------|---------|
| 黨鞭正式施壓（`ISSUE_PARTY_WHIP`） | 高 | 推動 public 向黨的方向靠近 |
| 選區民調驟降（`PUBLISH_POLL`） | 中-高 | 推動 public 向選區偏好靠近 |
| 媒體爆料醜聞（`LEAK_TO_MEDIA`） | 高 | 強制降低某議題能見度（沉默）|
| 私下交易達成（`NEGOTIATE_DEAL`） | 中 | 讓 hidden 局部更新（真心接受交換）|
| 同盟叛黨（觀察事件） | 中 | 重新評估聯盟可靠性，可能觸發 Reflect |
| 歷史承諾被查（媒體翻舊帳） | 高 | poignancy 爆高 → 立即觸發 Reflect |

---

### 資料模型（Convex schema 草稿）

```typescript
// 雙層立場存在 Agent 資料中
stances: defineTable({
  agentId,
  hidden:  v.object({ economic: v.number(), environment: v.number(), ... }),
  public:  v.object({ economic: v.number(), environment: v.number(), ... }),
  backbone: v.number(),       // 0-1，來自 Politician-skill profile
  lastShift: v.number(),      // timestamp，用於時間衰減計算
})

// 立場偏移事件日誌（可追蹤、可視覺化）
stanceShiftLog: defineTable({
  agentId,
  triggeredBy: v.string(),    // 觸發事件 node_id
  dimension:   v.string(),    // "economic" | "environment" | ...
  delta:       v.number(),    // 偏移量
  rationalization: v.string(),// LLM 生成的合理化敘事
  timestamp:   v.number(),
})
```

---

## PoC 驗證場景

1. 給定一個有爭議的法案（如：最低工資調漲）
2. 放入 5 位不同立場的政治人物 Agent
3. 執行 3 輪辯論 + 投票
4. 檢查：投票結果是否符合各 Agent 的已知立場？
5. 修改一個 Agent 的選區民調壓力 → 重跑 → 觀察是否改票
6. **關鍵指標：** 行為一致性分數（預期立場 vs 實際投票）

### 多面性專項驗證

7. 給同一個 Agent 在「黨團密室」vs「院會記名投票」各跑一次相同情境
8. 觀察：兩個場合的發言內容與投票是否出現落差？
9. 觸發媒體爆料事件 → 檢查 Agent 是否生成合理化 thought 節點
10. **關鍵指標：** `hidden_stance` 與 `public_stance` 的偏差軌跡是否符合直覺
