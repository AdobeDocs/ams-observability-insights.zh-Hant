---
title: 效能
description: 改善「可觀察性深入分析」中的查詢、圖表和控制面板效能。
source-git-commit: 143a9567da06f0137ef690795cfbc41bda1780ca
workflow-type: tm+mt
source-wordcount: '227'
ht-degree: 3%
---

## 效能 {#performance}

每個查詢都會以15秒逾時和位元組讀取預算執行。 此檔案說明可觀察性深入分析如何自動加速查詢，以及如何設計在任何時間範圍內保持快速的控制面板。

### 統計加速（自動）

`Transaction`個圖表讀取預先彙總資料表，當這些資料的&#x200B;**全部**&#x200B;保留時：

| 條件 | 需求 |
| ----------------- | ------------------------------------------------------------------------------------------------------------------------ |
| 形狀 | 查詢有`TIMESERIES` （數字/表格總計一律讀取原始資料以判斷準確性） |
| 視窗 | ≥ 3 h （1分鐘階層） · ≥ 1 d （5分鐘階層） · ≥ 7 d （1小時階層） |
| 彙總 | 僅限`count(*)`、`percentage(*, WHERE status = 'Error')`、`apdex(duration, 500)`、`percentile(duration, 50 \| 95 \| 99)` |
| `FACET` / `WHERE` | 僅於`service`，`appInstance`，`kind` |

`CpuUtilization`和`MemoryUtilization` (`average(value)`)和`JvmCpuRecentUtilization` (`average(value)`)圖表，其中`FACET host` / `WHERE host …`超過≥3小時會讀取1分鐘的統計。

**設計秘訣：**&#x200B;對於7至31天的趨勢Widget，請遵循適合統計的設定（p95，錯誤%，Apdex @ 500，計數，FACET服務） — 在任何範圍都能保持快速。 針對較短的視窗或資料表儲存`FACET host`/`name`、`average(duration)`或`urlPath`篩選器。

### 一般提示

- 在每個面向的查詢上使用`LIMIT` — 尤其是`FACET urlPath`、`traceId`、`jcr.path` （高基數）。
- 在低基數屬性(`service`、`kind`、`status`)上透過`WHERE`提早篩選。
- 當您只需要傳入要求時，偏好使用`Transaction`而非`Span` — 列數少得多。
- `uniqueCount`是精確的；在長視窗的高基數屬性上，它是最昂貴的彙總。
- 避免使用`TIMESERIES MAX`，除了幾分鐘的視窗。
- 儀表板Widget快取結果30秒；重新整理按鈕會強制重新執行。
- 點選15秒逾時或位元組讀取預算錯誤的Widget — 縮小視窗、新增篩選器或減少Facet。
