---
title: 疑難排解
description: 疑難排解Observability Insights中的SQL查詢、圖表和控制面板。
source-git-commit: 873e7b9f279c3276fc0197969d667d40d022968a
workflow-type: tm+mt
source-wordcount: '335'
ht-degree: 5%
---

## 疑難排解 {#troubleshooting}

當您&#x200B;**驗證**&#x200B;或&#x200B;**執行**&#x200B;查詢時，錯誤訊息會顯示在編輯器下方，並在儀表板上的Widget本身上顯示。 在第一欄中找到訊息。

| 訊息/症狀 | 原因 | 修正 |
| ---------------------------------------------------------------------------- | ------------------------------------------------------------------ | -------------------------------------------------------------------------------------------------------------------------------------- |
| `Unknown entity 'X'` | 拼寫錯誤或非目錄名稱 | 檢查[資料字典](#data-dictionary) /查詢指南面板。 |
| `Attribute 'x' is not available on Entity` | 屬性不在該實體上（例如`CpuUtilization`上的`duration`） | 追蹤屬性與量度屬性的比較 — 請參閱[資料字典](#data-dictionary)。 |
| `Raw attributes cannot be mixed with aggregates; use FACET to group results` | `SELECT service, count(*)` | `SELECT count(*) … FACET service`. |
| `FACET and TIMESERIES require an aggregate query` | 具有FACET/TIMESERIES的事件查詢 | 使用彙總。 |
| `FACET supports at most five attributes` | > 5個Facet | 縮小。 |
| `average requires a numeric attribute` | e.g. `average(service)` | 使用`uniqueCount`/`latest`或數值屬性。 |
| `rate requires TIMESERIES` / `rate is only available for counter measures` | `rate()`不含TIMESERIES或在量測軌上 | 新增`TIMESERIES AUTO`；使用✔計數器實體（[已組織的量度實體](../references/data-dictionary.md#curated-metric-entities)）；量測計使用`derivative()`。 |
| `rate cannot be combined with other aggregates` | 多個具有`rate()`的專案 | 每個Widget一個`rate()`；與公式結合。 |
| `percentOfTotal cannot be combined with TIMESERIES` | — | 移除時間序列。 |
| `apdex requires an entity with a status attribute` | 量度實體上的`apdex` | 使用`Transaction`/`Span`/`AppRequest`。 |
| `ORDER BY 'x' must reference a selected value or alias` | 依未選取的專案排序 | 依據別名對彙總和排序進行別名。 |
| `IN requires between 1 and 100 values` | — | 分割或使用`LIKE`。 |
| `Use IS NULL or IS NOT NULL for null comparisons` | `attr = NULL` | 使用`IS NULL`。 |
| `Unexpected character '{'` | 在新增Widget預覽中或儲存時`{{variable}}` | 請參閱[教學課程3](../tutorials/tutorial-3.md#make-dashboards-reusable-with-variables)限制。 |
| `Unexpected character ';'` / `'"'` | 分號或雙引號 | 移除`;`；使用單引號。 |
| `Expected a positive duration such as 30 minutes` | `TIMESERIES 5m` | 寫入`TIMESERIES 5 minutes`。 |
| `Time range cannot exceed 31 days` | 選取器> 31天 | 縮小視窗。 |
| `Select a customer before running a SQL query` | 無租使用者/示範租使用者 | 在租使用者切換器中挑選客戶。 |
| `All queries in a formula must be the same kind…` | 總計和時間序列或FACET的混合 | 將所有子查詢變更為純查詢或所有`TIMESERIES AUTO`；移除FACET。 |
| `Query 'x' must select exactly one metric…` | 子查詢有兩個彙總 | 每個子查詢一個彙總。 |
| `Unknown query name 'x' - must be one of: …` | 公式參考缺少的名稱 | 完全符合名稱（區分大小寫）。 |
| **儲存失敗** | Widget查詢在伺服器端無效（通常為`{{var}}`） | 開啟每個最近編輯的Widget並&#x200B;**執行**&#x200B;它；修正或移除無效的專案。 |
| _選取{{x}}_&#x200B;的值 | 變數沒有選取範圍 | 在變數列中挑選一個值。 |
| _過去……無活動_ | 視窗中沒有列 | 加寬視窗；使用探索查詢（[資料探索](../examples/query-cookbook.md#data-discovery)）驗證篩選器。 |
| 線條破碎為點 | 貯體小於資料步調 | 使用`TIMESERIES AUTO` （≥2分鐘）或長條圖。 |
| Facet顯示空字串群組 | 某些列遺失屬性 | 新增`WHERE attr != ''`。 |
| _此舊版SQL面板已不再執行_ | 舊的Raw-SQL產生器中的面板 | 在SQL中編輯→重建。 |
