# 產品定位聲明

> 版本：1 ｜ 更新日期：2026-04-30 ｜ 狀態：current
> 對應缺口：[../02-gaps.md §18](../02-gaps.md)、[../02-gaps.md §15](../02-gaps.md)、[../02-gaps.md §16](../02-gaps.md)
> 目的：釘死使用者族群，讓前端互動、倫理邊界、規則可配置性的設計有明確依據

---

## 定位

**研究者** + **遊戲開發者**，兩個族群並存。

---

## 兩個族群的需求差異

| 維度 | 研究者 | 遊戲開發者 |
|------|--------|-----------|
| **主要需求** | 情境注入、參數調整、可量化指標、情境分岔 | 可嵌入的 Agent 行為引擎、乾淨的 API 介面 |
| **前端互動** | 觀察 + 注入事件（Observer + Injector） | 最小前端，主要用 API / SDK 整合到自己的遊戲 |
| **政治人物** | 真實政治人物（Politician-skill 精確 profile） | 虛構人物為主，參數自訂 |
| **制度規則** | 特定國家規則（台灣立法院、美國國會） | 需要 rule-pack 可換，甚至完全自訂 |
| **倫理需求** | 需要明確 disclaimer（模擬 ≠ 預測） | 虛構人物 → 倫理風險低；若用真實人物則同研究者 |

---

## 對現有設計的影響

### #18 前端互動定位（解決）

兩個模式，不是二選一，是兩層介面：

```
Layer 1：Research Observatory（研究者用）
  → 議場視覺化（PixiJS 座位圖）
  → 即時 stance drift 圖表
  → 外部事件注入面板（接 external-event-model.md）
  → PoC 驗收指標儀表板（接 observability-model.md）

Layer 2：SDK / API（遊戲開發者用）
  → Convex mutation / query 作為 API 直接呼叫
  → Agent profile 可用自訂 JSON 替換 Politician-skill 輸出
  → rule-pack 切換（接 #16 規則可配置性）
  → 不依賴 Layer 1 的前端
```

研究者會用到 Layer 1 + Layer 2；遊戲開發者只需要 Layer 2。

### #16 規則可配置性（影響升高）

遊戲開發者需要 rule-pack，這讓 #16 從「可能要支援」變成「應該支援」。但 PoC 不需要，先把 `InstitutionalRules` 的常數集中到 `simulationConstants.ts`，之後再做 rule-pack 載入機制。

### #15 倫理邊界（分層處理）

```
研究者使用真實政治人物：
  → 輸出頁面加 disclaimer：「本模擬為研究工具，結果不代表真實人物立場或行為預測」
  → 不對外公開原始 LLM 輸出（避免被斷章取義）

遊戲開發者使用虛構人物：
  → 無特殊限制
  → 若使用者選擇嵌入真實政治人物 profile，自動套用上方 disclaimer 規則
```

Disclaimer 實作：在 `agents` schema 加 `isRealPerson: boolean`，所有輸出管道在 `isRealPerson = true` 時自動附加 disclaimer 字串。

---

## PoC 的定位

PoC 對準**研究者**族群：5 個真實政治人物 Agent、1 個法案場景、可量化驗收指標。  
遊戲開發者的 API / SDK 層在 PoC 之後設計。
