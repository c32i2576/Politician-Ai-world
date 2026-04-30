# 參考專案技術索引

> 版本：1 ｜ 更新日期：2026-04-30 ｜ 狀態：current
> 對應缺口：[02-gaps.md](02-gaps.md) #4 #7 #8 #9 #10 #11 #13 #14 #17 #18
> 目的：把三個本地參考專案的關鍵模組對應到未解缺口，方便實作時直接查閱原始碼

---

## 本地專案路徑對照

| 文件別稱 | GitHub | 本地路徑 | 語言／框架 |
|---------|--------|---------|-----------|
| Reverie（Generative Agents） | [joonspk-research/generative_agents](https://github.com/joonspk-research/generative_agents) | `D:\work\AI\generative_agents` | Python / Django |
| ai-town | [a16z-infra/ai-town](https://github.com/a16z-infra/ai-town) | `D:\work\AI\ai-town` | TypeScript / Convex / React |
| MiroFish | [666ghj/MiroFish](https://github.com/666ghj/MiroFish) | `D:\work\AI\MiroFish` | Python Flask + Vue 3 |

---

## 1. Generative Agents（Reverie）— 關鍵模組與缺口對應

```
D:\work\AI\generative_agents\
├── reverie\backend_server\
│   ├── reverie.py                          主模擬迴圈（fork 機制在此）
│   └── persona\
│       ├── persona.py                      Agent 入口：perceive/plan/reflect/execute
│       ├── memory_structures\
│       │   ├── scratch.py                  短期工作記憶（身份特質、當前動作）
│       │   ├── associative_memory.py       記憶流（ConceptNode + SPO 三元組）
│       │   └── spatial_memory.py           空間記憶樹
│       └── cognitive_modules\
│           ├── perceive.py                 感知（att_bandwidth / retention 過濾）
│           ├── retrieve.py                 記憶檢索（recency+relevance+importance）
│           ├── plan.py                     日程生成與分解
│           ├── reflect.py                  反思觸發（importance_trigger_max）
│           └── execute.py                  動作輸出
```

### 對應缺口

| 缺口 | 可借用的模組 | 借用重點 |
|------|------------|---------|
| **#8 Reflect 節流** | `reflect.py`、`scratch.py` | `importance_trigger_max` 累積閾值機制；`importance_trigger_curr` 遞減計數器；可加 per-agent cooldown 在同一邏輯上 |
| **#13 資訊可見性** | `perceive.py` | `att_bandwidth=3`（每 tick 最多感知 N 件事）+ `retention=5`（去重視窗）；政治版改為「場合 × 角色關係」矩陣 |
| **#17 可重現性** | `reverie.py` | fork 機制：`ReverieServer(fork_sim_code, new_sim_code)` 從存檔點分叉；隨機種子透過 Python `random.seed()` 注入 |
| **#14 派系動態** | `associative_memory.py` | SPO 三元組記錄盟友/衝突事件（`subject-predicate-object`）；poignancy 給衝突事件打高分，影響 retrieve 權重 |

---

## 2. ai-town — 關鍵模組與缺口對應

```
D:\work\AI\ai-town\
├── convex\
│   ├── engine\
│   │   ├── abstractGame.ts         引擎核心：loadWorld/handleInput/tick/saveWorld
│   │   └── gameLoop.ts             step runner（generation number 防 race condition）
│   ├── aiTown\
│   │   ├── main.ts                 世界 step 入口
│   │   ├── agent.ts                Agent tick：shouldStartOperation 判斷
│   │   ├── player.ts               Player 實體（位置 → 政治版改為立場向量）
│   │   ├── conversation.ts         2 人對話 → 政治版擴充為多人協商
│   │   └── world.ts                World 狀態文件（< 幾十 KB 約束在此）
│   └── agent\
│       ├── memory.ts               向量記憶：對話摘要 → embedding → Convex vector DB
│       └── conversation.ts         LLM 對話流程（internalAction 非同步呼叫）
└── src\
    ├── App.tsx                     React 入口 + Convex Provider
    └── components\
        └── Game.tsx                PixiJS 渲染層（議場視覺化的替換起點）
```

### 對應缺口

| 缺口 | 可借用的模組 | 借用重點 |
|------|------------|---------|
| **#9 容錯策略** | `abstractGame.ts`、`gameLoop.ts` | generation number：每個 world step 帶版本號，若 LLM operation 回傳時 generation 已變動則丟棄（天然防止 stale 寫入）；單一 step 失敗不卡整個引擎 |
| **#17 可重現性** | `gameLoop.ts` | 輸入佇列（inputs table）是唯一寫入管道，重放同一輸入序列可完全重現；LLM 呼叫加 `temperature=0` + `seed` 參數 |
| **#18 前端互動定位** | `agent.ts`、`conversation.ts`、`src/components/` | ai-town 的使用者只能「注入對話」（不能直接操控 Agent 決策）；政治模擬可沿用此模式 + 加「注入事件」通道（媒體爆料、外部危機） |
| **#4 選民層規模** | `world.ts`（World 大小約束） | 議員 ≤ 100 人完全符合；選民若做個體 Agent 需走統計聚合（poll state 取代個體），避免 World 超過幾十 KB |
| **#7 LLM 成本估算** | `agent.ts`（`shouldStartOperation`） | 只有「需要決策」的 tick 才呼叫 LLM（state machine check）；預估：100 議員 × 平均每 5 tick 決策一次 × 每次 1K tokens ≈ 每 session 20K calls；可在此閘控 |

---

## 3. MiroFish — 關鍵模組與缺口對應

```
D:\work\AI\MiroFish\
├── backend\app\
│   ├── api\
│   │   ├── graph.py                知識圖譜 API（文件 → Zep）
│   │   ├── simulation.py           模擬 API（啟動／狀態／停止）
│   │   └── report.py               報告 API（觸發 ReACT Agent）
│   └── services\
│       ├── graph_builder.py        文件解析 → Zep 圖譜建構
│       ├── ontology_generator.py   LLM 自動生成實體類型（改為政治實體）
│       ├── zep_tools.py            Zep Cloud 查詢工具（graph_query / insight_forge）
│       ├── oasis_profile_generator.py   Zep 圖譜 → OASIS Agent profile
│       ├── simulation_config_generator.py  LLM 從圖譜生成 simulation 參數
│       ├── simulation_runner.py    CAMEL OASIS subprocess 執行核心
│       ├── simulation_manager.py   背景任務狀態管理（Task 模型）
│       └── report_agent.py         ReACT：search_zep / insight_forge / interview
├── frontend\src\
│   ├── views\MainView.vue          5-step workflow 容器
│   └── components\
│       ├── Step1-5 各 .vue 檔
│       └── GraphPanel.vue          D3.js 互動圖譜（議場派系視覺化起點）
└── docs\simulation.md              OASIS 模擬流程文件
```

### 對應缺口

| 缺口 | 可借用的模組 | 借用重點 |
|------|------------|---------|
| **#10 冷啟動** | `graph_builder.py`、`zep_tools.py`、`oasis_profile_generator.py` | 完整的「文件 → 圖譜 → Agent profile」管線；政治版：把議員公開資料（質詢紀錄、法案共同提案、媒體報導）餵入 `graph_builder`，圖譜建立後即有初始關係網路，無需等模擬跑出來 |
| **#11 profile → Agent schema** | `oasis_profile_generator.py` | 實際程式碼範例：讀 Zep 圖譜 → 呼叫 LLM → 輸出 OASIS-compatible `personality`、`stance` JSON；政治版在此加 `hidden_stance`、`backbone`、`stanceVector` 欄位 |
| **#9 容錯策略** | `simulation_manager.py`、`simulation_runner.py` | `Task` 模型記錄狀態（PENDING/RUNNING/DONE/ERROR）；subprocess 超時強制終止；LLM 回傳格式錯誤有 fallback parse；可直接複用這套狀態機 |
| **#7 LLM 成本估算** | `simulation_runner.py` | OASIS 每次動作呼叫 LLM 一次；可在此加計數器估算 token 用量；MiroFish 的 `BOOST_LLM` 雙模型設計（高能力模型只給報告）也是成本控制手段 |
| **#14 派系動態** | `zep_tools.py`（`insight_forge`） | Zep 的關係圖 + `insight_forge` 可主動查詢「誰和誰有合作紀錄」；政治版在每 session close 時呼叫，計算派系凝聚度 |
| **#18 前端互動** | `GraphPanel.vue`（D3.js）、`MainView.vue` | MiroFish 的 Step 5 互動視覺化是「觀察者模式」（無法注入事件）；政治版在此基礎上加「注入事件面板」，使用者可觸發媒體爆料或外部危機 |

---

## 4. 跨專案整合建議（針對最高優先缺口）

### 冷啟動管線（#10）— 直接複用 MiroFish

```
議員公開資料（PDF/網頁）
    ↓ MiroFish graph_builder.py（文件解析 → Zep 圖譜）
    ↓ MiroFish ontology_generator.py（實體：議員/黨/委員會/法案）
    ↓ MiroFish oasis_profile_generator.py（圖譜 → Agent profile JSON）
    ↓ 加入 hidden_stance / backbone 欄位（Politician-skill 六軌輸出）
    → 初始 Convex agents 表（ai-town 引擎接管後續）
```

### Reflect 節流（#8）— Reverie 邏輯移植到 Convex

```typescript
// 在 convex/politicalWorld/agent.ts 的 tick() 中
if (agent.importanceTriggerCurr <= 0) {
  agent.startOperation(ctx, internal.agent.politicianOps.reflect, {
    agentId: agent.id,
    cooldownTicks: 3,   // per-agent 冷卻（Reverie 原版無此）
  });
  agent.importanceTriggerCurr = IMPORTANCE_TRIGGER_MAX;
}
// 每個新記憶節點寫入時遞減 importanceTriggerCurr
// 參考：generative_agents/reverie/backend_server/persona/cognitive_modules/reflect.py
```

### 可重現性（#17）— 兩層種子

| 層次 | 做法 | 參考 |
|------|------|------|
| 輸入序列 | Convex `inputs` table 是唯一寫入管道，保存完整 replay log | ai-town `abstractGame.ts` |
| LLM 採樣 | `temperature=0` + `seed` 參數（OpenAI / Claude API 均支援） | — |
| Fork 分析 | 從特定 tick 的 World snapshot 建立新 simulation（複製 Convex 表） | Reverie `reverie.py` fork 機制 |

---

## 5. 尚未有參考實作的缺口

下列缺口在三個參考專案中均**無直接對應**，需從頭設計：

| 缺口 | 說明 |
|------|------|
| **#4 選民層行為模型** | 三個專案的 Agent 都是同質的；政治版需要「議員（精確 profile）+ 選民（統計聚合）」雙層分離，選民以 `opinionState` 表取代個體 Agent |
| **#5 PoC 可量化指標** | 需自行定義：行為一致性分數計算公式、Reflect 品質評分方法 |
| **#6 ground truth 對標** | 需決定：台灣立法院歷史投票紀錄作為 baseline；LLM 重現率計算方式 |
| **#12 灰色地帶動作集** | MiroFish 只有合法社群動作；Reverie 只有日常生活動作；需自行定義政治灰色地帶動作及制度應對規則 |
| **#15 倫理責任邊界** | 三個專案都沒有針對真實政治人物模擬的 disclaimer 機制 |
| **#16 規則可配置性** | 需設計 rule-pack 介面，讓台灣立院／美國國會規則可切換 |
