# Politician-skill 整合合約

> 版本：1 ｜ 更新日期：2026-04-30 ｜ 狀態：current
> 對應缺口：[../02-gaps.md §11](../02-gaps.md)
> 來源參考：`d:\work\AI\Politician-skill\politicians\lai-ching-te\meta.json`、`graph/nodes.json`、`graph/edges.json`
> 對齊：[bill-model.md](bill-model.md)（stanceVector 維度）、[../01-research.md L375-L413](../01-research.md#L375-L413)（雙層立場 + 壓力函數）
> 目的：釘死 Politician-skill 輸出到模擬 Agent 初始化的完整轉換規則

---

## 0. 問題陳述

[01-research.md L21](../01-research.md#L21) 說「六軌研究輸出即為 profile 來源」，  
[01-research.md L388](../01-research.md#L388) 用了 `backbone: 0.3`，  
但兩者之間的轉換完全未定義。

本文件回答三個問題：
1. Politician-skill 的哪些欄位對應模擬的哪些 schema 欄位？
2. `backbone` 怎麼從六軌研究資料推導？
3. 兩個 repo 之間的 interface contract 是什麼（版本更新時怎麼辦）？

---

## 1. Politician-skill 的輸出結構（現況）

```
politicians/<slug>/
  meta.json          ← 結構化 JSON（版本、spectrum、evidence counts）
  political.md       ← 5 心智模型 + 10 決策啟發式（非結構化文本）
  persona.md         ← 句式指紋、口癖、風格光譜（非結構化文本）
  limitations.md     ← 做不到的事（非結構化文本）
  research/          ← 六軌原始語料（六個 .md 檔）

graph/
  nodes.json         ← 所有政治人物節點（含 spectrum 數值）
  edges.json         ← 人際關係邊（含 weight -1~+1）
```

**注意：meta.json 與 nodes.json 的 spectrum 尺度不一致。**

| 來源 | `economic` 值（賴清德） | 尺度 |
|------|----------------------|------|
| `meta.json` | `2.5` | -5 ~ +5（推測） |
| `graph/nodes.json` | `-0.2` | -1 ~ +1 |

**本合約統一以 `graph/nodes.json` 的 -1~+1 為標準**，meta.json 的 spectrum 欄位僅供參考。若 Politician-skill 日後統一尺度，本文件更新映射係數即可。

---

## 2. 模擬需要的維度（Canonical Dimensions）

bill-model.md 已固定 `stanceVector` 為三個 **政策維度**：

```typescript
{ economic: number, environment: number, social: number }
```

Agent 的 `hidden` / `public` stance 需要覆蓋這三個維度，才能做 `stanceVector · agentStance` 的點積計算（自然投票傾向）。

額外加入兩個 **政治特質維度**，用於壓力函數和派系計算：

```typescript
type StanceDimensions = {
  // 政策維度（與 bill stanceVector 對齊）
  economic:     number,   // -1=左傾 / +1=右傾
  environment:  number,   // -1=開發優先 / +1=環保優先
  social:       number,   // -1=保守 / +1=進步

  // 政治特質維度
  authority:    number,   // -1=反建制 / +1=威權
  unification:  number,   // -1=獨立 / +1=統一（台灣特有）
}
```

**為什麼只有 5 個維度而不是 Politician-skill 的全部？**

Politician-skill 的 `hawkish`、`populism`、`us_china_align`、`taiwan_identity` 在 PoC 階段沒有對應的法案 stanceVector，加進來只會增加未被驗證的噪音。等 PoC 後視需要擴充。

---

## 3. 欄位映射表（Politician-skill → Agent Schema）

### 3.1 stance 初始化

| 目標欄位 | 來源 | 轉換規則 |
|---------|------|---------|
| `hidden.economic` | `nodes.json spectrum.economic` | 直接用（已 -1~+1） |
| `hidden.environment` | `nodes.json spectrum.environmentalism` | 若為 null → 預設 0.0 |
| `hidden.social` | 無直接對應 | 從 `authority`（反向）+ `populism` 推導（見 §3.3） |
| `hidden.authority` | `nodes.json spectrum.authority` | 直接用 |
| `hidden.unification` | `nodes.json spectrum.unification` | 直接用；非台灣政治人物 → 0.0 |
| `public.*` | 同 `hidden.*` | **初始化時 public = hidden**（無歷史壓力紀錄） |
| `backbone` | 六軌研究推導 | 見 §3.2 |

### 3.2 backbone 推導公式

`backbone` 代表「抵抗壓力的能力」，Politician-skill 沒有直接欄位，從三個來源推導：

```python
def derive_backbone(politician_slug: str) -> float:
    # 來源 1：黨紀偏離率（最重要，權重 0.6）
    #   從 research/02-votes-decisions.md 計算：
    #   偏離次數 / 總投票次數（只算有黨紀立場的法案）
    party_deviation_rate = count_party_line_breaks(slug) / total_whip_votes(slug)
    # 資料不足（< 5 次）→ 使用 DEFAULT_BACKBONE

    # 來源 2：authority 維度（反向，低威權 = 更獨立，權重 0.3）
    authority = nodes_spectrum(slug, "authority")  # -1~+1
    independence_score = (-authority + 1) / 2       # 線性映射到 0~1

    # 來源 3：style tag 偵測（修正項，權重 0.1）
    style_tags = meta_json(slug, "tags.style")
    style_bonus = 0.1 if "強斷言" in style_tags else 0.0
    style_penalty = -0.1 if "共識導向" in style_tags else 0.0

    backbone = (
        0.6 * party_deviation_rate
        + 0.3 * independence_score
        + 0.1 * (0.5 + style_bonus + style_penalty)  # 中位 0.5 ± 修正
    )
    return round(clip(backbone, 0.1, 0.9), 2)

DEFAULT_BACKBONE = 0.4  # 資料不足時的保守預設
```

**為什麼上限是 0.9 而非 1.0？**  
backbone = 1.0 表示完全不受任何壓力影響，現實中任何政治人物都不會達到。0.9 作為可達上限留有餘地。

### 3.3 `hidden.social` 推導（無直接欄位）

Politician-skill 沒有「社會保守/進步」維度，從兩個相關維度估算：

```python
def derive_social_stance(slug: str) -> float:
    authority = nodes_spectrum(slug, "authority")   # 高威權 → 偏保守
    populism  = nodes_spectrum(slug, "populism")    # 高民粹 → 依選民，台灣選民偏進步

    # 簡單加權，預設 0 表示「資訊不足，中立」
    if authority is None and populism is None:
        return 0.0

    social = (-0.6 * (authority or 0)) + (0.4 * (populism or 0))
    return round(clip(social, -1.0, 1.0), 2)
```

這個推導是有損的，預計誤差大。**優先在 PoC 後用真實法案投票資料校正**，初始值只是合理起點。

---

## 4. 關係圖初始化（Zep / 派系層）

```python
# edges.json 轉換為 Zep 初始關係
for edge in edges_json:
    zep.add_relationship(
        from_agent = edge["from"],
        to_agent   = edge["to"],
        rel_type   = edge["rel"],       # ally / rival / mentor / protege ...
        trust_score = (edge["weight"] + 1) / 2,  # -1~+1 → 0~1
        evidence    = edge["evidence"],
    )
```

`trust_score` 正規化到 0~1 對應壓力函數裡的 `coalition_need * trust_score`（[01-research.md L406](../01-research.md#L406)）。

**注意**：edges.json 的關係是「已知的公開關係」，初始化到 Zep 後代表的是模擬開始時的 **歷史起點**，不是 hidden 關係。模擬過程中建立的關係（私下談判、密室交易）才是動態更新的部分。

---

## 5. 非結構化欄位的使用方式

`political.md`（心智模型）和 `persona.md`（表達 DNA）是非結構化文本，不進入 Convex schema，而是作為：

```typescript
agents: defineTable({
  // ...
  skillMd:     v.string(),   // politicians/<slug>/SKILL.md 的完整內容
  // 或（較省 token）：
  politicalMd: v.string(),   // political.md 內容
  personaMd:   v.string(),   // persona.md 內容
})
```

每次 Agent 的 `decideVote()` / `negotiate()` LLM 呼叫時，將這些內容注入 system prompt。

**為什麼不解析成結構？**  
心智模型和啟發式規則的價值在於 LLM 能完整理解上下文，強行結構化（如把「安全優先發展論」拆成 5 個欄位）反而損失語意。只有數值化的維度（stance、backbone）才需要進 DB schema。

---

## 6. 完整 Agent 初始化 Schema

```typescript
// 從 Politician-skill 初始化一個 Agent 時，需要寫入以下表：

// 1. agents 表
{
  slug:        "lai-ching-te",          // 對應 Politician-skill 的 slug
  name:        "賴清德",
  party:       "dpp",
  role:        "president",
  schemaVersion: "2.0",                  // meta.json schema_version
  researchCutoff: "2026-04-25",          // meta.json research_cutoff
  skillMd:     "<完整 SKILL.md 內容>",
  profileHash: "sha256(<SKILL.md>)",     // 版本追蹤用（見 §7）
}

// 2. stances 表
{
  agentId:   <上方的 agentId>,
  hidden: {
    economic:    -0.2,   // 來自 nodes.json
    environment:  null → 0.0,
    social:      derived,
    authority:   -0.6,
    unification: -0.9,
  },
  public: { /* 初始 = hidden */ },
  backbone:  0.52,        // 推導值（見 §3.2）
  lastShift: <init_tick>,
}

// 3. Zep（edges.json 轉為初始關係）
//    見 §4
```

---

## 7. 版本更新策略

Politician-skill 更新某位政治人物的研究時（新增語料、調整心智模型），模擬世界裡**正在運行的 Agent** 如何處理：

```
                  ┌──────────────────────┐
                  │ Politician-skill 更新 │
                  └──────────┬───────────┘
                             ↓
              profileHash 是否變更？
               ↙                    ↘
             否                      是
   （無需處理）         ┌──────────────────────────┐
                       │ 模擬是否正在進行中？        │
                       └────────┬─────────────────┘
                       ↙                  ↘
                     否                    是
              直接更新 skillMd            等下一個
              + 更新 backbone            session close
              + 更新 stances             後再更新
              （不清空 Zep 歷史）          （不中斷進行中的 tick）
```

**原則：**
- `skillMd`（非結構化）可以隨時更新，LLM 下次呼叫自動生效
- `stances`（數值）只在 session 邊界更新，避免「法案審議到一半立場突然變了」
- Zep 的動態關係**不覆蓋**，只補充初始關係中新增的 edge

---

## 8. 本合約未覆蓋的欄位（需另行來源）

| 欄位 | 說明 | 建議來源 |
|------|------|---------|
| `party.whip_strength` | 黨的紀律強度，非個人屬性 | 建立 parties 表，手動設定（KMT 高、DPP 中、TPP 低） |
| `donor_interest` | 金主方向，Politician-skill 不研究 | PoC 暫時省略此壓力來源 |
| `risk_aversion` | 對媒體曝光的規避程度 | 從 `authority` 正向映射暫代（authority 高 → 較保守）|
| `constituency_base` | 選區 ID | 從 `profile.position` 人工對應 |

---

## 9. PoC 最小實作

| 步驟 | 內容 | 預估工時 |
|------|------|---------|
| 1 | 寫 `politician_to_agent.py`：讀 meta.json + nodes.json → 輸出 agents + stances 初始值 | 1 天 |
| 2 | 實作 `derive_backbone()`（先用黨紀偏離率，資料不足用 DEFAULT_BACKBONE） | 0.5 天 |
| 3 | 實作 `derive_social_stance()`（先用 authority 推導，PoC 後校正） | 0.5 天 |
| 4 | 用賴清德跑一次，人工驗證產出的 stances 數值是否合理 | 含在 PoC 工時內 |
| 5 | edges.json → Zep 初始化 | 0.5 天 |

**先砍掉**：
- `donor_interest` 壓力來源（PoC 不需要）
- `social` 維度校正（先用推導值，PoC 後再補）
- 版本更新策略（PoC 期間不會有 Politician-skill 升版）

---

## 10. 主要 Tradeoff 與兜底

| Tradeoff | 解法 |
|---------|------|
| nodes.json 與 meta.json 尺度不一致 | 統一用 nodes.json（-1~+1）；meta.json spectrum 欄位凍結不用 |
| backbone 推導公式依賴 research/02-votes-decisions.md 解析 | 資料不足時 fallback 到 DEFAULT_BACKBONE = 0.4 |
| `social` 無直接來源，推導有損 | 明確標記為「推導值」；PoC 後以投票紀錄校正 |
| skillMd 體積大（18KB/人）× 100 議員 = 1.8MB | 個別 Agent 的 LLM 呼叫只注入自己的 skillMd，不是全部 |
| Zep 初始關係覆蓋動態關係 | 初始化用單獨的 `source: "politician-skill-init"` tag，與模擬動態關係區分 |

---

## 11. 待決事項

- [ ] `social` 維度長期是否要加入 Politician-skill 的六軌研究（作為新的標準輸出）？還是永遠從 authority 推導？
- [ ] `backbone` 推導中的 style tag 清單（「強斷言」「共識導向」）需要 Politician-skill 那邊定義規範化 tag 集合
- [ ] bill-model 的 `stanceVector` 目前只有 economic / environment / social；若未來加入 `authority` / `unification` 維度，需同步更新 stanceVector schema
- [ ] 非台灣政治人物（無 `unification` 維度）的 fallback 值是否要建立 per-country 維度子集？
- [ ] `schemaVersion` 版本不相容時（如 Politician-skill 升到 3.0 大改 spectrum 欄位），需要 migration script，目前未設計
