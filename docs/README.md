# 文件索引

> 更新日期：2026-04-29

## 閱讀順序

新進入這份專案先照這個順序讀：

1. [01-research.md](01-research.md) — 架構研究原文（總覽、分層、認知迴圈、壓力函數、PoC 場景）
2. [02-gaps.md](02-gaps.md) — 缺口分析（18 項待補項，每項標註「已解決／未解決」）
3. `design/` — 已釘下的子系統設計（每個子系統一份檔，當前版號）
4. `archive/` — 被取代的歷史版本（只看交叉引用時才需要）

---

## 文件地圖

```
docs/
├── README.md                       本檔（索引）
├── 01-research.md                  架構研究原文（v1）
├── 02-gaps.md                      缺口分析（v1）
├── design/                         子系統設計（current 版本）
│   ├── bill-model.md               法案資料模型 v1
│   ├── time-model.md               時間／會期模型 v2（事件驅動）
│   └── media-model.md              媒體與輿論層 v2（三通道）
└── archive/                        歷史版本（已被取代）
    └── design-priority-three.md    v1 三合一設計檔（§1 抽出至 bill-model；§2 §3 已被 v2 取代）
```

---

## 缺口解決狀態快查

對照 [02-gaps.md](02-gaps.md)：

| # | 缺口 | 狀態 | 對應檔 |
|---|------|------|--------|
| 1 | 法案資料模型 | ✅ 已解決 | [design/bill-model.md](design/bill-model.md) |
| 2 | 時間／會期模型 | ✅ 已解決 | [design/time-model.md](design/time-model.md) |
| 3 | 媒體 Agent 與輿論層 | ✅ 已解決 | [design/media-model.md](design/media-model.md) |
| 4 | 選民層 | ⏳ 未解決 | — |
| 5 | PoC 關鍵指標 | ⏳ 未解決 | — |
| 6 | ground truth／對標 | ⏳ 未解決 | — |
| 7 | LLM 成本估算 | ⏳ 未解決 | — |
| 8 | Reflect 節流 | ⏳ 未解決 | — |
| 9 | 容錯策略 | ⏳ 未解決 | — |
| 10 | 冷啟動 | ⏳ 未解決 | — |
| 11 | profile → Agent schema | ⏳ 未解決 | — |
| 12 | 灰色地帶動作集 | ⏳ 未解決 | — |
| 13 | 資訊可見性模型 | ⏳ 未解決 | — |
| 14 | 派系動態 | ⏳ 未解決 | — |
| 15 | 倫理／責任邊界 | ⏳ 未解決 | — |
| 16 | 規則可配置性 | ⏳ 未解決 | — |
| 17 | 可重現性／隨機種子 | ⏳ 未解決 | — |
| 18 | 前端互動定位 | ⏳ 未解決 | — |

---

## 命名與版號規則

- **`NN-name.md`**：頂層的階段性文件，數字代表閱讀順序，不是版號
- **`design/<system>-model.md`**：子系統當前版本（無 v 字樣，當前即唯一）
- **檔頭 `> 版本：N`**：每份檔開頭用 blockquote 標版號、更新日期、狀態（current / archived / superseded）
- **取代舊檔**：舊檔搬到 `archive/`，新檔在開頭引用舊檔位置；不在檔名上掛 `-v2`
- **缺口索引**：所有設計檔的「對應缺口」一律指向 `02-gaps.md`，不指向其他設計檔
