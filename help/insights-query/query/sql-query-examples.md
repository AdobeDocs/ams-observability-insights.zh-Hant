---
title: SQL查詢範例
description: 探索Observability Insights遙測的SQL查詢範例。
source-git-commit: 4ad45ae5cbb2a7b348fdad1cd2681a72be12f5eb
workflow-type: tm+mt
source-wordcount: '311'
ht-degree: 10%
---

## SQL查詢範例

以下是更完整的範例。 對於此查詢：

- 您選擇`Transaction`作為實體。
- 您使用`SELECT`來計算第95個百分位數持續時間。
- 您使用`FACET`依服務將結果分組。
- 您使用`TIMESERIES AUTO`在自動調整大小的間隔內繪製資料。

```sql
FROM Transaction SELECT percentile(duration, 95) AS p95 FACET service TIMESERIES AUTO
```

這會在折線圖上為每個服務產生一行，預設顯示10個最繁忙的服務。

### 傳入請求的基本查詢

+++檢視查詢

所有傳入要求的平均回應時間（以毫秒為單位）：

```sql
SELECT average(duration) FROM Transaction
```

+++

### 包含點的屬性名稱

+++檢視查詢

點狀名稱不需要引號：

```sql
SELECT count(*) FROM Transaction WHERE http.statusCode >= 500 FACET http.route
```

+++

### 查詢傳回多個欄

+++檢視查詢

一個查詢中的數個彙總會成為表格中的欄：

```sql
SELECT count(*) AS requests, average(duration) AS avgMs, percentile(duration, 95) AS p95, max(duration) AS maxMs FROM Transaction FACET service
```

+++

### 一次的條件式彙總

+++檢視查詢

總計、錯誤和5xx並排：

```sql
SELECT count(*) AS total, count(*) FILTER (WHERE status = 'Error') AS errors, count(*) FILTER (WHERE http.statusCode >= 500) AS http5xx FROM Transaction FACET service
```

+++

### 查詢已組織的量度

+++檢視查詢

一段時間內每個主機的平均CPU：

```sql
SELECT average(value) AS cpu FROM CpuUtilization FACET host TIMESERIES AUTO
```

+++

### 依名稱查詢任何OpenTelemetry測量結果

+++檢視查詢

對沒有已組織實體的量度使用`MetricGauge` / `MetricSum`：

```sql
SELECT latest(value) AS bytesUsed FROM MetricSum WHERE metric = 'jvm.memory.used' FACET host, pool
```

+++

<a id="what-sql-doesnt-do"></a>

## SQL不會執行的動作

SQL故意很小。 這可讓查詢保持安全、快速且與租使用者隔離。 若您需要其中一項功能，請使用替代方案。

| 功能 | 狀態 | 替代方案 |
| ------------------------------------------- | ------------- | -------------------------------------------------------------------------------------------------------------------------------------- |
| `SELECT *` | 不支援 | 列出您想要的屬性： `SELECT timestamp, name, duration, traceId …`。 |
| 混合原始屬性和彙總 | 不支援 | 使用`FACET attr`或`latest(attr)` / `earliest(attr)`。 |
| `SELECT`中的數學(`a - b`， `count(*) / 60`) | 不支援 | **公式** Widget （控制面板）。 |
| 聯結和子查詢 | 不支援 | **公式** Widget，或兩個並排的Widget。 |
| `HAVING` | 不支援 | `ORDER BY alias` + `LIMIT`. |
| `COMPARE WITH` | 不支援 | 變更時間選擇器，或將兩個Widget放在不同的索引標籤上。 |
| `FACET CASES` | 不支援 | 每個案例一個`count(*) FILTER (WHERE …)`欄（請參閱[教學課程3](../../tutorials/tutorial-3.md#group-values-into-cases-with-filter)）。 |
| `histogram()` | 不支援 | 分段`FILTER`欄（請參閱[教學課程3](../../tutorials/tutorial-3.md#build-a-latency-histogram)）。 |
| `uniques()` / `keyset()` | 不支援 | `FACET attr LIMIT n`列出不同的值。 探索查詢會列出量度和填入的屬性。 |
| `RLIKE` （規則運算式） | 不支援 | 具有`%`和`_`萬用字元的`LIKE` / `NOT LIKE`，或`IN (…)`。 |
| 評論(`--`， `//`， `/* */`) | 不支援 | 將備註放入Widget標題或&#x200B;**文字** Widget。 |
| `SINCE`中的絕對日期 | 不支援 | 使用時間選擇器的絕對範圍。 |
| 型別轉換(`numeric()`， `string()`) | 不需要 | 屬性是在目錄中輸入。 |
