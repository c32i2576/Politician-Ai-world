# 架構研究：不足之處分析

> 版本：1 ｜ 更新日期：2026-04-29 ｜ 狀態：current
> 對應文件：[01-research.md](01-research.md)
> 目的：列出現行架構研究的設計盲點與待補項，作為進入實作前的 checklist

---

## 一、設計上的硬缺口（會卡住實作）

### 1. 法案本身的資料模型缺失
動作集列出 `INTRODUCE_BILL`、`AMEND_BILL`，但法案的結構從未定義：
- 條文格式
- 版本控制與修正案家族
- 委員會路徑
- 表決階段（一讀／二讀／三讀）

法案是引擎要動起來的最核心物件，比 stance schema 還重要，但 [架構分層](01-research.md#L76-L113) 與 [Convex schema 草稿](01-research.md#L484-L503) 都沒列。

> **狀態：已解決** — 見 [design/bill-model.md](design/bill-model.md)（條文結構化 + `stanceVector` 投影 + `expiresAtSession` 綁 session id）。

### 2. 時間／會期模型未定義
文件多次出現「每輪」、「會期」、「tick 1 Hz」，但三者怎麼對應？
- 一個 tick = 幾分鐘現實時間？
- 會期切換的觸發條件是什麼？
- 選舉週期如何介入？

直接影響 stance 的 `time_decay` 公式（[L411](01-research.md#L411) 提到但未定義）和 reflect 觸發頻率。

> **狀態：已解決** — 見 [design/time-model.md](design/time-model.md)（事件驅動 session + per-topic λ 衰減 + tick 不綁實時）。

### 3. 媒體 Agent 與輿論層完全缺席
動作集裡有 `LEAK_TO_MEDIA`、`PUBLISH_POLL`、`RUN_AD_CAMPAIGN`，但：
- 媒體 Agent 怎麼決定報導什麼？
- 輿論值怎麼生成？
- `media_spotlight` 的數值從哪來？

[壓力函數](01-research.md#L396-L413) 用了 `media_spotlight` 變數但沒有產生來源。

> **狀態：已解決** — 見 [design/media-model.md](design/media-model.md)（三通道架構 + `apply_news_to_opinion` + `get_spotlight` 釘死壓力函數源頭）。

### 4. 「選民層」只在概念上存在
[L128](01-research.md#L128) 提到「程序性生成的選民」與議員雙層分開處理，但下列細節都沒寫：
- 選民 Agent 的行為模型
- 規模與聚合方式（個體還是統計分布？）
- 與議員的耦合介面

---

## 二、評估與驗收（沒有它就無法判斷成功）

### 5. PoC 的關鍵指標未定義
- [L514](01-research.md#L514) 寫「行為一致性分數（預期立場 vs 實際投票）」
- [L521](01-research.md#L521) 寫「偏差軌跡是否符合直覺」

兩者都沒有計算方式、閾值、判定為通過／失敗的標準。「符合直覺」尤其無法落地驗收。

> **狀態：已解決** — 見 [design/observability-model.md](design/observability-model.md)（`behavior_consistency_score` 門檻 ≥ 0.70；`stance_drift_analysis` 以 high_backbone < low_backbone 為通過條件）。

### 6. 缺少 ground truth／對標方案
- 是否打算用真實立法院／國會的歷史投票紀錄反測？
- Agent 重跑真實法案，預期吻合率多少算成功？

沒談這個就是純自圓其說。

---

## 三、工程可行性盲點

### 7. LLM 成本／延遲完全沒估算
- Reverie 的五步認知迴圈 × N 個 Agent × 每 tick 執行 → token 與延遲量級可能無法承受
- 100 議員、tick 1 Hz、每 tick 多次 LLM 呼叫，每天會燒多少？
- 選項 C（PoC）裡甚至沒列「成本驗證」這項

> **狀態：已解決** — 見 [design/llm-cost-model.md](design/llm-cost-model.md)（A/B/C 呼叫分級 + 具體成本估算 + 降級策略）。

### 8. Reflect 沒有節流機制
[L221-L229](01-research.md#L221-L229) 直接照搬 Reverie 的「重要性閾值觸發」。問題：
- 政治場景高張力時 importance 會爆
- 每個 Agent 同時 reflect 會把成本拉到不可預期
- 需要 per-agent 冷卻或全局節流

> **狀態：已解決** — 見 [design/llm-cost-model.md](design/llm-cost-model.md)（三層節流：per-agent cooldown + global cap + 優先佇列）。

### 9. 無效動作／無限循環的容錯策略
- LLM 回傳語法錯誤的動作怎麼辦？
- 引用不存在的法案 ID 怎麼辦？
- Agent 卡死循環怎麼辦？
- 單一 World single-threaded 下，一個 Agent 失敗會不會卡住整個 tick？

### 10. 冷啟動問題
- 第一次模擬時所有 Agent 之間沒有 Zep 關係、沒有對話歷史
- 「派系自動形成」會退化成純立場向量距離
- 需要從 Politician-skill 的研究結果初始化關係網路，但介面未定義

---

## 四、與 Politician-skill 的銜接

### 11. profile → Agent 的具體 schema 缺失
- [L21](01-research.md#L21) 寫「六軌研究輸出即為 profile 來源」
- [L388](01-research.md#L388) 用了 `backbone: 0.3`
- 但 Politician-skill 怎麼產出 `backbone`、`hidden_stance` 各維度數值？

方法論、信度、欄位對照表都沒寫。這是兩個專案的整合點，不能模糊。

> **狀態：已解決** — 見 [design/politician-skill-contract.md](design/politician-skill-contract.md)（欄位映射表 + backbone 推導公式 + 版本更新策略）。

---

## 五、行為模型的窄化

### 12. 動作集只覆蓋陽光面
缺了下列灰色地帶動作：
- 賄賂
- 勒索
- 利益輸送
- 人事威脅
- 造假民調
- 椿腳動員

真實政治模擬若沒這些會嚴重失真。要嘛明確宣告「不模擬」，要嘛補進動作集與制度規則的應對。

### 13. 資訊可見性模型缺失
有 `hidden_stance` / `public_stance` 是好開始，但缺：
- 「密室談判內容怎麼不洩漏給其他 Agent」
- 「誰能 perceive 到誰的什麼動作」
- 統一的 visibility 規則

Reverie 用 `vision_r` 處理空間可見性，政治場景需要的是「角色關係 + 場合」的可見性矩陣。

### 14. 派系動態過度簡化
[L68-L72](01-research.md#L68-L72) 「立場相近自動形成聯盟」忽略：
- 歷史恩怨
- 世代
- 地域
- 利益分配

真實派系不只是立場聚類。

---

## 六、跨橫切議題（容易被忽略）

### 15. 倫理與責任邊界
模擬真實在世政治人物，有名譽風險、被當成「預測」誤用的風險。文件完全沒提：
- disclaimer
- 輸出內容的責任歸屬
- 是否限定虛構人物或加入扭曲層

這在發布前一定會被問到。

### 16. 在地化／規則可配置性
- 台灣立法院、美國國會、英國下議院的議事規則差異巨大
- `InstitutionalRules` 寫死常數（[L52-L57](01-research.md#L52-L57)）等於只能模擬一種制度
- 要不要支援 rule-pack 切換？

### 17. 可重現性／隨機種子
- LLM 採樣本身有隨機性
- 加上 tick 順序、reflect 觸發，重跑同一組輸入結果會不同
- 實驗對比（[PoC 第 5 步](01-research.md#L513)「修改民調壓力 → 重跑 → 觀察」）需要種子控制方案
- 文件沒談

### 18. 前端互動定位不清
[L110-L112](01-research.md#L110-L112) 列了三個視覺化元件，但使用者是：
- 被動觀察？
- 可以注入事件？
- 扮演某個 Agent？

這影響整個 input pipeline 設計。

> **狀態：已解決** — 見 [design/product-positioning.md](design/product-positioning.md)。兩層介面：研究者用 Research Observatory（觀察 + 注入）；遊戲開發者用 SDK/API 層（不依賴前端）。

### 19. 外部事件注入介面缺失

架構圖的所有動作都由 Agent 內部產生。但研究者需要：
- 注入情境事件（世界事件 → 觀察 Agent 反應）
- 在測試時直接覆蓋 world state（修改民調壓力、強制法案推進）

兩條管道性質不同，混用會讓「自然發展」與「研究者介入」無法區分，實驗無法重現。

> **狀態：已解決** — 見 [design/external-event-model.md](design/external-event-model.md)（雙通道 + `processedAtTick` 稽核紀錄）。

---

## 建議優先補的三項

進入實作前，建議優先釘下下列三項；其餘可以留到 PoC 後再補。

1. **法案 + 時間模型 + 媒體層**
   沒這三個引擎跑不起來。

2. **PoC 的可量化驗收指標**
   沒這個無法判斷階段是否完成。

3. **LLM 成本估算與 reflect 節流策略**
   決定選 A／B／C／D 的關鍵變量。

---

## 待辦索引（對照 01-research.md 章節）

| 章節 | 不足項編號 |
|------|----------|
| [政治平台動作集](01-research.md#L39-L46) | #12 |
| [制度規則引擎](01-research.md#L48-L57) | #16 |
| [動態派系形成](01-research.md#L68-L72) | #14 |
| [整體架構分層](01-research.md#L76-L113) | #1, #2, #18 |
| [Generative Agents 認知迴圈](01-research.md#L142-L170) | #7, #8 |
| [三層記憶結構](01-research.md#L174-L208) | #13 |
| [Reflect 模組](01-research.md#L211-L232) | #8 |
| [實作路徑選項](01-research.md#L338-L367) | #7 |
| [雙層立場架構](01-research.md#L375-L391) | #11 |
| [壓力函數](01-research.md#L394-L413) | #2, #3 |
| [Convex schema 草稿](01-research.md#L482-L503) | #1 |
| [PoC 驗證場景](01-research.md#L506-L522) | #5, #6, #10, #17 |
| 全域（未涵蓋） | #4, #9, #15 |
