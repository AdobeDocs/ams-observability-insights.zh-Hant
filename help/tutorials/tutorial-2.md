---
title: 教學課程2：控制您的資料
description: 瞭解如何在可觀察性深入分析中篩選、分組和控制查詢結果。
source-git-commit: 143a9567da06f0137ef690795cfbc41bda1780ca
workflow-type: tm+mt
source-wordcount: '361'
ht-degree: 0%
---

## 教學課程2：控制您的資料 {#tutorial-2-control-your-data}

在本教學課程中，您將學習將原始計數轉換為答案的函式和子句：相異計數、最新值、百分比、百分位數、萬用字元、排序和多維度分組。

**在本教學課程中：** [7.1 uniqueCount](#71-count-distinct-values-with-uniquecount) · [7.2 latest/earliest](#72-find-the-latest-and-earliest-values) · [7.3百分比](#73-calculate-percentages) · [7.4百分位數](#74-understand-latency-with-percentile) · [7.5標籤](#75-make-results-readable-with-labels) · [7.6萬用字元和清單](#76-match-patterns-and-lists) · [7.7排序](#77-sort-and-rank-with-order-by) · [7.8多面向](#78-group-by-several-attributes) · [7.9過去的windows](#79-look-at-a-past-window)

### 以`uniqueCount`計算不同的值

每個服務有多少台主機和不同的追蹤？

```sql
SELECT uniqueCount(host) AS hosts, uniqueCount(traceId) AS traces FROM Transaction FACET service
```

`uniqueCount`為精確值，而非估計值。

若要&#x200B;**列出**&#x200B;不同的值本身（NRQL呼叫的是`uniques()`），在屬性上要面向：

```sql
SELECT count(*) AS requests FROM Transaction FACET host LIMIT 100
```

### 尋找最新和最早的值

`latest(attr)`會傳回最近的值，而`earliest(attr)`會傳回最舊的值。 它們對「目前狀態」表格非常有用：

```sql
SELECT latest(value) AS currentCpu FROM CpuUtilization FACET host
```

```sql
SELECT earliest(timestamp) AS firstSeen, latest(timestamp) AS lastSeen FROM Transaction FACET service
```

### 計算百分比

`percentage(*, WHERE condition)`會傳回符合條件的資料列共用。 這是錯誤率：

```sql
SELECT percentage(*, WHERE status = 'Error') AS errorPct FROM Transaction
```

依服務分析趨勢：

```sql
SELECT percentage(*, WHERE status = 'Error') AS errorPct FROM Transaction FACET service TIMESERIES AUTO LIMIT 10
```

並追蹤緩慢請求的份額：

```sql
SELECT percentage(*, WHERE duration > 2000) AS slowPct FROM Transaction FACET service
```

### 透過`percentile`瞭解延遲

平均值會隱藏離群值。 百分位數顯示大部分使用者體驗的內容(p50)以及不幸者體驗的內容(p95、p99)：

```sql
SELECT percentile(duration, 50) AS p50, percentile(duration, 95) AS p95, percentile(duration, 99) AS p99 FROM Transaction TIMESERIES AUTO
```

>[!TIP]
>
> `Transaction.duration`的p50、p95和p99透過長視窗的統計提供，因此每週的檢視都能保持快速。

### 讓結果可透過標籤讀取

針對包含空格的標籤，請使用帶有單引號的`AS`：

```sql
SELECT count(*) AS 'Requests', percentile(duration, 95) AS 'p95 (ms)' FROM Transaction FACET service
```

>[!NOTE]
>
> 別名僅適用於`SELECT`個專案。 Facet欄一律標示屬性名稱。

### 比對模式和清單

`LIKE`將`%`用於任何數目的字元，將`_`用於唯一一個字元：

```sql
SELECT count(*) AS requests FROM Transaction WHERE urlPath LIKE '/content/%.html' FACET urlPath LIMIT 20
```

排除具有`NOT LIKE`的靜態資產：

```sql
SELECT count(*) AS requests FROM Transaction WHERE urlPath NOT LIKE '%.js' AND urlPath NOT LIKE '%.css' AND urlPath NOT LIKE '%.png' FACET urlPath LIMIT 20
```

比對具有`IN`和`NOT IN`的清單（最多100個值）：

```sql
SELECT count(*) AS requests FROM Transaction WHERE http.method IN ('POST', 'PUT', 'DELETE') FACET service
```

>[!NOTE]
>
> sql沒有規則運算式(`RLIKE`)。 將`LIKE`模式與`OR`結合，或使用`IN`。

### 使用`ORDER BY`排序和排名

依預設，多面結果會依第一個彙總排序。 使用`ORDER BY`來依其他專案排名，例如p95的最慢作業：

```sql
SELECT count(*) AS requests, percentile(duration, 95) AS p95 FROM Transaction FACET name ORDER BY p95 DESC LIMIT 20
```

或是先使用最少服務：

```sql
SELECT count(*) AS requests FROM Transaction FACET service ORDER BY requests ASC LIMIT 10
```

>[!TIP]
>
> 沒有`HAVING`。 若要只尋找「忙碌」作業，請依`requests`排名並保留前N位，或新增可移除雜訊的`WHERE` （例如`WHERE http.method != 'OPTIONS'`）。

### 依數個屬性分組

`FACET`最多接受五個屬性。 每個組合會變成一列或數列：

```sql
SELECT count(*) AS requests FROM Transaction WHERE http.statusCode >= 400 FACET service, http.method, http.statusCode LIMIT 30
```

### 檢視過去的視窗

沒有`COMPARE WITH`。 在儀表板和探索中，變更&#x200B;**時間選擇器**。 當您呼叫不含明確範圍的API時，`SINCE … UNTIL …`會提供您任何過去的視窗：

```sql
SELECT count(*) AS lastWeekSameHour FROM Transaction SINCE 169 hours ago UNTIL 168 hours ago
```
