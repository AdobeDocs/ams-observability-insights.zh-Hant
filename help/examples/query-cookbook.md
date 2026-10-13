---
source-git-commit: 263ba9e9b3eb1c6e4dd253ca03bd0ce8954c588b
workflow-type: tm+mt
source-wordcount: '565'
ht-degree: 0%
---
## 查詢逐步指南 {#query-cookbook}

逐步指南是現成可用的查詢資料庫，依主題分組。 每個查詢前面都有一個最適合它的圖表型別，以&#x200B;**粗體**&#x200B;表示。 時間子句會被省略，因為時間選擇器會控制視窗。

**在此檔案中：** [10.1資料探索](#101-data-discovery) · [10.2輸送量](#102-throughput) · [10.3延遲](#103-latency) · [10.4錯誤](#104-errors) · [10.5 Apdex](#105-apdex--user-experience) · [10.6服務和主機](#106-services-hosts--instances) · [10.7相依性](#107-downstream-dependencies) · [10.8 AEM](#108-aem-adobe-experience-manager) · [10.9相依性基礎架構](#109-infrastructure) · [10.10 JVM](#1010-jvm--runtime)

&#x200B;> [!TIP]
> 將範例值（例如`'checkout'`或`'web-publish-01'`）取代為您租使用者的實際值。 以下的資料探索查詢會列出這些查詢。

### 資料探索

**表格** — 哪些服務報告流量：

```sql
SELECT count(*) AS requests, uniqueCount(host) AS hosts FROM Transaction FACET service LIMIT 200
```

**資料表** — 哪些主機/階層存在：

```sql
SELECT count(*) AS requests FROM Transaction FACET hostTierPrefix, host LIMIT 200
```

**表格** — 哪些環境和地區存在：

```sql
SELECT count(*) AS spans FROM Span FACET environment, region LIMIT 50
```

**表格** — 每個量測量度報告，及其最新值：

```sql
SELECT count(*) AS samples, latest(value) AS latestValue FROM MetricGauge FACET metric, unit LIMIT 200
```

**表格** — 每個總和/計數器量度：

```sql
SELECT count(*) AS samples, latest(value) AS latestValue FROM MetricSum FACET metric, unit LIMIT 200
```

**資料表** — 已填入哪些span屬性（AEM範例）：

```sql
SELECT count(*) FILTER (WHERE sling.script != '') AS slingScripts, count(*) FILTER (WHERE jcr.operation != '') AS jcrOps, count(*) FILTER (WHERE db.system != '') AS dbCalls, count(*) FILTER (WHERE graphql.operation != '') AS graphql FROM Span FACET service LIMIT 50
```

### 輸送量

**數字** — 要求總數：

```sql
SELECT count(*) AS requests FROM Transaction
```

**行** — 輸送量趨勢：

```sql
SELECT count(*) AS requests FROM Transaction TIMESERIES AUTO
```

**區域** — 依服務的輸送量，前8名：

```sql
SELECT count(*) AS requests FROM Transaction FACET service TIMESERIES AUTO LIMIT 8
```

**列** — 透過HTTP方法的輸送量：

```sql
SELECT count(*) AS requests FROM Transaction FACET http.method LIMIT 10
```

**資料表** — 最忙碌的URL路徑：

```sql
SELECT count(*) AS requests, percentile(duration, 95) AS p95 FROM Transaction FACET urlPath ORDER BY requests DESC LIMIT 50
```

**數字** — 不同的追蹤：

```sql
SELECT uniqueCount(traceId) AS traces FROM Span
```

### 延遲

**數字** — p95回應時間（單位自動=毫秒）：

```sql
SELECT percentile(duration, 95) AS p95 FROM Transaction
```

**線** — p50 / p95 / p99在一個圖表上：

```sql
SELECT percentile(duration, 50) AS p50, percentile(duration, 95) AS p95, percentile(duration, 99) AS p99 FROM Transaction TIMESERIES AUTO
```

**行** — 每個服務的p95，最繁忙的10大：

```sql
SELECT percentile(duration, 95) AS p95 FROM Transaction FACET service TIMESERIES AUTO LIMIT 10
```

**列** — p95最慢的作業：

```sql
SELECT percentile(duration, 95) AS p95 FROM Transaction FACET name ORDER BY p95 DESC LIMIT 20
```

**資料表** — 每個作業的延遲設定檔：

```sql
SELECT count(*) AS calls, average(duration) AS avgMs, percentile(duration, 50) AS p50, percentile(duration, 95) AS p95, percentile(duration, 99) AS p99, max(duration) AS maxMs, stddev(duration) AS stddevMs FROM Transaction FACET name ORDER BY p95 DESC LIMIT 50
```

**列** — 每個作業總逗留時間的份額（時間預算所在處）：

```sql
SELECT percentOfTotal(duration) AS timeShare FROM Transaction FACET name LIMIT 20
```

**資料表** — 每個作業之單一最慢要求的追蹤識別碼：

```sql
SELECT count(*) AS calls, max(duration) AS slowestMs, latest(traceId, duration) AS slowestTraceId FROM Transaction FACET name ORDER BY slowestMs DESC LIMIT 50
```

**Number** — 共用低於2秒的要求：

```sql
SELECT percentage(*, WHERE duration > 2000) AS slowShare FROM Transaction
```

**表格** — 透過篩選計數（長條圖替代項）的延遲值區：

```sql
SELECT count(*) FILTER (WHERE duration <= 100) AS under100ms, count(*) FILTER (WHERE duration > 100 AND duration <= 500) AS to500ms, count(*) FILTER (WHERE duration > 500 AND duration <= 2000) AS to2s, count(*) FILTER (WHERE duration > 2000) AS over2s FROM Transaction FACET service LIMIT 20
```

### 錯誤次數

**數字** — 錯誤率%：

```sql
SELECT percentage(*, WHERE status = 'Error') AS errorRate FROM Transaction
```

**行** — 每個服務的錯誤率%：

```sql
SELECT percentage(*, WHERE status = 'Error') AS errorRate FROM Transaction FACET service TIMESERIES AUTO LIMIT 10
```

**行** — 錯誤計數趨勢：

```sql
SELECT count(*) AS errors FROM Transaction WHERE status = 'Error' TIMESERIES AUTO
```

**圓形** — HTTP狀態類別組合：

```sql
SELECT count(*) AS requests FROM Transaction WHERE http.statusCode IS NOT NULL FACET http.statusCode LIMIT 15
```

**表格** — 5xx熱點：

```sql
SELECT count(*) AS errors5xx FROM Transaction WHERE http.statusCode >= 500 FACET service, urlPath, http.statusCode LIMIT 50
```

**資料表** — 例外狀況型別的錯誤：

```sql
SELECT count(*) AS errors, uniqueCount(traceId) AS traces, latest(statusMessage) AS lastMessage FROM Span WHERE status = 'Error' FACET service, error.type LIMIT 50
```

**資料表** — 錯誤與總計並排：

```sql
SELECT count(*) AS total, count(*) FILTER (WHERE status = 'Error') AS errors, percentage(*, WHERE status = 'Error') AS errorPct FROM Transaction FACET service ORDER BY errors DESC LIMIT 20
```

**資料表** — 最新的失敗要求（事件清單）：

```sql
SELECT timestamp, service, name, urlPath, http.statusCode, duration, traceId FROM Transaction WHERE status = 'Error' LIMIT 100
```

### Apdex和使用者體驗

**Number** — Apdex （T = 500毫秒）：

```sql
SELECT apdex(duration, 500) AS apdex FROM Transaction
```

**行** — 每個服務的相加值：

```sql
SELECT apdex(duration, 500) AS apdex FROM Transaction FACET service TIMESERIES AUTO LIMIT 10
```

**長條** — 依URL路徑最差的Apdex （較嚴格的300毫秒目標）：

```sql
SELECT apdex(duration, 300) AS apdex, count(*) AS requests FROM Transaction WHERE urlPath LIKE '/content/%' FACET urlPath ORDER BY apdex ASC LIMIT 20
```

### 服務、主機和執行個體

**表格** — 服務計分卡：

```sql
SELECT count(*) AS requests, percentage(*, WHERE status = 'Error') AS errorPct, percentile(duration, 95) AS p95, apdex(duration, 500) AS apdex, uniqueCount(host) AS hosts FROM Transaction FACET serviceKey ORDER BY requests DESC LIMIT 50
```

**表格** — 每個執行個體（AppRequest包含消費者）：

```sql
SELECT count(*) AS requests, percentage(*, WHERE status = 'Error') AS errorPct, percentile(duration, 95) AS p95 FROM AppRequest FACET appInstance ORDER BY requests DESC LIMIT 50
```

**行** — 每個主機的輸送量：

```sql
SELECT count(*) AS requests FROM Transaction FACET host TIMESERIES AUTO LIMIT 10
```

**列** — 負載平衡檢查（每個主機的請求共用）：

```sql
SELECT count(*) AS requests FROM Transaction FACET host LIMIT 30
```

### 下游相依性

**資料表** — 已呼叫下游服務：

```sql
SELECT count(*) AS calls, percentile(duration, 95) AS p95, percentage(*, WHERE status = 'Error') AS errorPct FROM Span WHERE kind = 'Client' AND peer.service != '' FACET service, peer.service LIMIT 50
```

**資料表** — 呼叫的外部網域：

```sql
SELECT count(*) AS calls, percentile(duration, 95) AS p95, count(*) FILTER (WHERE status = 'Error') AS errors FROM Span WHERE kind = 'Client' AND externalDomain != '' FACET externalDomain ORDER BY calls DESC LIMIT 50
```

**長條** — 系統和操作的資料庫延遲：

```sql
SELECT count(*) AS calls, percentile(duration, 95) AS p95 FROM Span WHERE db.system != '' FACET db.system, db.operation LIMIT 30
```

**資料表** — 緩慢的資料庫名稱空間：

```sql
SELECT count(*) AS calls, average(duration) AS avgMs, percentile(duration, 99) AS p99 FROM Span WHERE db.system != '' FACET db.system, db.namespace ORDER BY p99 DESC LIMIT 30
```

**行** — 每個目的地的傳訊輸送量：

```sql
SELECT count(*) AS messages FROM Span WHERE messaging.system != '' FACET messaging.destination TIMESERIES AUTO LIMIT 10
```

**資料表** — 每個主題的製作者與消費者：

```sql
SELECT count(*) FILTER (WHERE kind = 'Producer') AS produced, count(*) FILTER (WHERE kind = 'Consumer') AS consumed, percentile(duration, 95) AS p95 FROM Span WHERE messaging.system != '' FACET messaging.system, messaging.destination LIMIT 30
```

**資料表** — RPC方法：

```sql
SELECT count(*) AS calls, percentile(duration, 95) AS p95, count(*) FILTER (WHERE status = 'Error') AS errors FROM Span WHERE rpc.system != '' FACET rpc.system, rpc.method LIMIT 30
```

### AEM (Adobe Experience Manager)

**區域** — 依AEM層級的流量（作者/發佈/Dispatcher前置詞）：

```sql
SELECT count(*) AS requests FROM Transaction WHERE hostTierPrefix != '' FACET hostTierPrefix TIMESERIES AUTO LIMIT 10
```

**行** — 發佈層p95：

```sql
SELECT percentile(duration, 95) AS p95 FROM Transaction WHERE host LIKE '%publish%' FACET host TIMESERIES AUTO LIMIT 10
```

**行** — 作者階層p95：

```sql
SELECT percentile(duration, 95) AS p95 FROM Transaction WHERE host LIKE '%author%' FACET host TIMESERIES AUTO LIMIT 10
```

**資料表** — 頁面要求(`/content`)與servlet呼叫(`/bin`)的比較：

```sql
SELECT count(*) FILTER (WHERE urlPath LIKE '/content/%') AS pages, count(*) FILTER (WHERE urlPath LIKE '/bin/%') AS servlets, count(*) FILTER (WHERE urlPath LIKE '/etc.clientlibs/%') AS clientlibs, count(*) FILTER (WHERE urlPath LIKE '/content/dam/%') AS assets FROM Transaction FACET hostTierPrefix LIMIT 20
```

**資料表** — 最慢的`/bin` servlet：

```sql
SELECT count(*) AS calls, percentile(duration, 95) AS p95, percentage(*, WHERE status = 'Error') AS errorPct FROM Transaction WHERE urlPath LIKE '/bin/%' FACET urlPath ORDER BY p95 DESC LIMIT 30
```

**資料表** — Sling要求URI：

```sql
SELECT count(*) AS requests, percentile(duration, 95) AS p95 FROM Transaction WHERE sling.requestUri != '' FACET sling.requestUri ORDER BY requests DESC LIMIT 100
```

**表格** — 慢速Sling指令碼：

```sql
SELECT count(*) AS executions, percentile(duration, 95) AS p95, sum(duration) AS totalMs FROM Span WHERE sling.script != '' FACET sling.script ORDER BY totalMs DESC LIMIT 50
```

**列** — 依時間共用的Sling servlet：

```sql
SELECT percentOfTotal(duration) AS timeShare FROM Span WHERE sling.servlet != '' FACET sling.servlet LIMIT 20
```

**資料表** — 慢速JSP：

```sql
SELECT count(*) AS executions, percentile(duration, 95) AS p95 FROM Span WHERE jsp.path != '' FACET jsp.path ORDER BY p95 DESC LIMIT 50
```

**列** — JCR操作組合：

```sql
SELECT count(*) AS operations FROM Span WHERE jcr.operation != '' FACET jcr.operation LIMIT 20
```

**表格** — 最熱的JCR路徑：

```sql
SELECT count(*) AS operations, percentile(duration, 95) AS p95, sum(duration) AS totalMs FROM Span WHERE jcr.path != '' FACET jcr.operation, jcr.path ORDER BY totalMs DESC LIMIT 50
```

**表格** — GraphQL (headless)作業：

```sql
SELECT count(*) AS calls, percentile(duration, 95) AS p95, count(*) FILTER (WHERE status = 'Error') AS errors FROM Span WHERE graphql.operation != '' FACET graphql.operation ORDER BY calls DESC LIMIT 50
```

**行** — 每個主機的Dispatcher請求率：

```sql
SELECT rate(value) AS requestsPerSecond FROM DispatcherRequestRate FACET host TIMESERIES AUTO
```

**資料表** — 交易的組成專案（區段劃分；設定單位`ms`）：

```sql
SELECT sum(callCount) AS calls, average(avgDurationMs) AS avgMs FROM SpanBreakdown WHERE rootOperation LIKE 'GET /content%' FACET rootOperation, segment ORDER BY calls DESC LIMIT 50
```

### 基礎架構

**行** — 每個主機的CPU：

```sql
SELECT average(value) AS cpu FROM CpuUtilization FACET host TIMESERIES AUTO
```

**行** — CPU使用者時間：

```sql
SELECT average(value) AS userPct FROM CpuUserPercent TIMESERIES AUTO
```

&#x200B;> [!NOTE]
> SQL會讀取每個查詢的一個實體，因此user/system/iowait/steeve需要四個並排的Widget （請參閱[blueprint 11.6](#116-infrastructure-capacity-board) W7-W10）。 公式會將其摺疊為一個數字，而不是四個系列。

**列** — 依主機的CPU p95：

```sql
SELECT percentile(value, 95) AS cpuP95 FROM CpuUtilization FACET host LIMIT 30
```

**資料表** — 主機健康情況快照：

```sql
SELECT average(value) AS avgCpu, max(value) AS peakCpu, latest(value) AS nowCpu FROM CpuUtilization FACET host ORDER BY peakCpu DESC LIMIT 50
```

**行** — 已使用的記憶體%：

```sql
SELECT average(value) AS memory FROM MemoryUtilization FACET host TIMESERIES AUTO
```

**行** — 記憶體成長率（洩漏搜尋）：

```sql
SELECT derivative(value) AS memoryTrend FROM MemoryUtilization FACET host TIMESERIES AUTO
```

**資料表** — 每次掛載使用的磁碟：

```sql
SELECT latest(value) AS usedPct FROM DiskUtilization FACET host, mountpoint ORDER BY usedPct DESC LIMIT 50
```

**數字** — 主機可用根磁碟的最低值（位元組值；設定單位`bytes`）：

```sql
SELECT min(value) AS freeBytes FROM DiskFreeBytes
```

**行** — 磁碟讀取與寫入輸送量（每個都一個）：

```sql
SELECT rate(value) AS readBytesPerSec FROM DiskReadRate FACET host TIMESERIES AUTO
```

```sql
SELECT rate(value) AS writeBytesPerSec FROM DiskWriteRate FACET host TIMESERIES AUTO
```

**線路** — 網路接收/傳輸：

```sql
SELECT rate(value) AS rxBytesPerSec FROM NetworkReceiveRate FACET host TIMESERIES AUTO
```

```sql
SELECT rate(value) AS txBytesPerSec FROM NetworkTransmitRate FACET host TIMESERIES AUTO
```

**行** — 網路錯誤：

```sql
SELECT rate(value) AS rxErrors FROM NetworkReceiveErrorRate FACET host TIMESERIES AUTO
```

**行** — 平均載入：

```sql
SELECT average(value) AS load5m FROM LoadAverage5m FACET host TIMESERIES AUTO
```

**表格** — CPU和記憶體排名最前的處理序：

```sql
SELECT average(value) AS cpuPct, max(value) AS peakCpuPct FROM ProcessCpuUtilization FACET host, process.name ORDER BY cpuPct DESC LIMIT 30
```

```sql
SELECT average(value) AS memMb FROM ProcessMemoryUsage FACET host, process.name ORDER BY memMb DESC LIMIT 30
```

**表格** — 任何依名稱的原始量測計（不需要組織實體）：

```sql
SELECT average(value) AS avgValue, max(value) AS maxValue FROM MetricGauge WHERE metric = 'system.cpu.utilization' FACET host, state LIMIT 50
```

### JVM和執行階段

**行** — 每個主機使用的棧積(MB)：

```sql
SELECT average(value) AS heapMb FROM JvmHeapUsed FACET host TIMESERIES AUTO
```

**資料表** — 已使用的棧積與限制（兩個Widget或公式 — 請參閱[Blueprint 11.7](#117-jvm-health-board)）：

```sql
SELECT latest(value) AS heapLimitMb FROM JvmHeapLimit FACET host LIMIT 50
```

**行** — JVM執行緒：

```sql
SELECT latest(value) AS threads FROM JvmThreadCount FACET host TIMESERIES AUTO
```

**行** — JVM處理序CPU：

```sql
SELECT average(value) AS jvmCpu FROM JvmCpuRecentUtilization FACET host TIMESERIES AUTO
```

**資料表** — JVM記憶體集區（一般總和量度）：

```sql
SELECT latest(value) AS bytesUsed FROM MetricSum WHERE metric = 'jvm.memory.used' FACET host, pool LIMIT 100
```
