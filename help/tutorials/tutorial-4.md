---
title: 教學課程4：進階函式
description: 瞭解如何使用進階SQL函式來分析Observability Insights遙測。
source-git-commit: 143a9567da06f0137ef690795cfbc41bda1780ca
workflow-type: tm+mt
source-wordcount: '507'
ht-degree: 1%
---

## 教學課程4：進階函式 {#tutorial-4-advanced-functions}

本教學課程涵蓋支援更深入調查的功能和實體：變異性、成長性、「最差範例」查閱、總計份額、交易構成和任意OpenTelemetry量度。

**在本教學課程中：** [9.1 stddev](#91-measure-consistency-with-stddev) · [9.2衍生](#92-spot-growth-with-derivative) · [9.3排名最新](#93-find-the-worst-example-with-ranked-latest) · [9.4 percentOfTotal](#94-find-where-time-goes-with-percentoftotal) · [9.5 SpanBreakdown](#95-break-down-transactions-with-spanbreakdown) · [9.6任何量度](#96-discover-and-query-any-metric) · [9.7選擇實體](#97-choose-the-right-trace-entity) · [9.8巢狀彙總](#98-work-around-nested-aggregation)

### 與`stddev`的量值一致性

兩個作業可以共用一個平均值，但行為仍然非常不同。 標準差顯示延遲的變化程度：

```sql
SELECT count(*) AS requests, average(duration) AS avgMs, stddev(duration) AS stddevMs FROM Transaction FACET name ORDER BY stddevMs DESC LIMIT 20
```

大於平均值的標準差會指向偶爾非常緩慢請求的操作。 接下來看看它的p99。

### 使用`derivative`的點成長

`derivative(attr)`會傳回值區之間屬性平均值的每秒變更。 在尋找正在攀爬的量測計時（例如洩漏前的棧積使用量）時，此外掛程式非常有用：

```sql
SELECT derivative(value) AS heapMBPerSec FROM JvmHeapUsed FACET host TIMESERIES 5 minutes
```

持續的正數線表示成長。 負值表示量測軌在下降（例如，在記憶體回收之後）。 類似`rate`，`derivative`需要`TIMESERIES`，且必須是唯一的彙總。

### 尋找排名為`latest`的最差範例

`latest`和`earliest`的第二個引數將「最近」變更為「此屬性的最高值」。 `latest(traceId, duration)`傳回最慢要求的追蹤ID，所以您可以將其貼入追蹤搜尋：

```sql
SELECT count(*) AS requests, max(duration) AS slowestMs, latest(traceId, duration) AS slowestTrace, latest(urlPath, duration) AS slowestPath FROM Transaction FACET name ORDER BY slowestMs DESC LIMIT 15
```

使用`earliest(traceId, duration)`擷取最快的範例以進行比較。

### 尋找時間與`percentOfTotal`的關聯

`percentOfTotal(attr)`顯示每個群組在總數中所佔的份額。 套用至`duration`時，會顯示哪些作業耗用最多伺服器時間。 這通常比哪些作業最慢更有用：

```sql
SELECT count(*) AS requests, sum(duration) AS totalMs, percentOfTotal(duration) AS shareOfServerTime FROM Transaction FACET name LIMIT 15
```

`percentOfTotal`無法與`TIMESERIES`結合。

### 劃分與`SpanBreakdown`的交易

`SpanBreakdown`在內嵌時預先彙總。 它會記錄每個交易(`rootOperation`)所花費的時間（以分鐘為單位），每個子作業(`segment`)。 它可回答「此端點在做什麼？」 便宜：

```sql
SELECT sum(callCount) AS calls, average(avgDurationMs) AS avgMs FROM SpanBreakdown WHERE rootOperation LIKE '%GET%' FACET rootOperation, segment LIMIT 25
```

分析一個區段在一段時間內的趨勢：

```sql
SELECT average(avgDurationMs) AS avgMs FROM SpanBreakdown FACET segment TIMESERIES AUTO LIMIT 8
```

>[!NOTE]
>
> `avgDurationMs`未使用單位自動標籤。 將Widget **Unit**&#x200B;設定為`ms`。

### 探索及查詢任何量度

SQL沒有`keyset()`或`SHOW EVENT TYPES`。 而是列出報告的每個量度，以及其單位和最新值：

```sql
SELECT count(*) AS samples, latest(value) AS latestValue FROM MetricGauge FACET metric, unit LIMIT 200
```

```sql
SELECT count(*) AS samples, latest(value) AS latestValue FROM MetricSum FACET metric, unit LIMIT 200
```

然後依名稱查詢任何量度。 例如，每個集區使用的JVM記憶體：

```sql
SELECT latest(value) AS bytesUsed FROM MetricSum WHERE metric = 'jvm.memory.used' FACET host, pool LIMIT 50
```

若要檢視每個服務填入了哪些&#x200B;**span屬性**，請計算非空白值：

```sql
SELECT count(*) FILTER (WHERE http.route != '') AS withRoute, count(*) FILTER (WHERE db.system != '') AS dbCalls, count(*) FILTER (WHERE peer.service != '') AS withPeer, count(*) FILTER (WHERE error.type != '') AS withErrorType FROM Span FACET service LIMIT 50
```

>[!TIP]
>
> 當已組織的實體存在時（例如`CpuUtilization`），會偏好使用它而非`MetricGauge`。 它會套用正確的篩選器和單位轉換，而且可能是向上彙整加速。

### 選擇正確的追蹤實體

| 使用 | 當您想要時 | 範例 |
| ------------- | ------------------------------------------------------------------------------------------- | --------------------------------- |
| `Transaction` | 傳入HTTP/RPC要求KPI （輸送量、延遲、錯誤、Apdex）。 加速統計。 | 黃金訊號 |
| `AppRequest` | 相同的KPI，**包含訊息消費者** （Kafka和類似專案） | 事件導向服務 |
| `Span` | 請求以下的任何內容：傳出HTTP、DB、傳訊、AEM Sling/JCR/JSP、內部跨距 | 相依性和程式碼路徑分析 |

計算每個範圍種類，以檢視檢測的內容：

```sql
SELECT count(*) AS spans, percentile(duration, 95) AS p95 FROM Span FACET kind
```

外部網域傳出呼叫：

```sql
SELECT count(*) AS calls, percentage(*, WHERE status = 'Error') AS errorPct, percentile(duration, 95) AS p95 FROM Span WHERE kind = 'Client' AND externalDomain != '' FACET externalDomain LIMIT 20
```

### 解決巢狀彙總的問題

NRQL可讓您彙總具有子查詢的彙總，例如「每主機最大值的平均值」。 SQL不會。 以下是替代方案：

| 目標 | SQL方法 |
| --------------------------------------- | ---------------------------------------------------------- |
| 依群組的值，然後為群組排名 | `FACET group` + `ORDER BY alias` + `LIMIT` |
| 兩個彙總的比率 | [公式Widget](#86-combine-queries-with-formula-widgets) |
| 每個主機的值，然後是「最差主機」 | `FACET host ORDER BY alias DESC LIMIT 1` |
| 一段時間內每個主機的平均值 | `FACET host TIMESERIES AUTO`，並讀取序列 |

範例：最差主機依p95：

```sql
SELECT percentile(duration, 95) AS p95 FROM Transaction FACET host ORDER BY p95 DESC LIMIT 1
```
