---
source-git-commit: 263ba9e9b3eb1c6e4dd253ca03bd0ce8954c588b
workflow-type: tm+mt
source-wordcount: '1311'
ht-degree: 1%
---
## 儀表板Blueprint {#dashboard-blueprints}

每個Blueprint都是完整、複雜的儀表板。 它列出目標、索引標籤和區段配置，以及每個具有其圖表型別、單位、臨界值、大小和查詢的Widget。 在12欄格線上，大小為`width×height`。 從上到下逐一建置，然後拖曳加以調整。

| 藍圖 | 客群 | 反白顯示 |
| ------------------------------------------------------------------------------------ | --------------------- | ---------------------------------------------------------------------------------- |
| [11.1 Golden Signals / SLO](#111-golden-signals--slo-board) | SRE、隨選 | 具有臨界值的KPI圖磚、Apdex、錯誤%、飽和度 |
| [11.2錯誤分類](#112-error-triage-board) | 隨選 | 錯誤類別、5xx熱點、具有追蹤ID的失敗請求 |
| [11.3延遲深入分析](#113-latency-deep-dive-board) | 效能工程師 | 百分位數、延遲貯體、變異性、交易組成、最慢追蹤 |
| [11.4下游相依性](#114-downstream-dependencies-board) | 服務擁有者 | 輸出HTTP、DB、傳訊、RPC |
| [11.5 AEM作者/發佈/ Dispatcher](#115-aem-author--publish--dispatcher-board) | AEM團隊 | 每層區段、Sling/JSP/JCR、Dispatcher載入 |
| [11.6基礎結構容量](#116-infrastructure-capacity-board) | 平台/容量 | CPU模式，記憶體，磁碟，網路速率，公式 |
| [11.7 JVM健康狀態](#117-jvm-health-board) | Java團隊 | 棧積%公式，每個主機和集區的棧積，執行緒， JVM CPU |
| [11.8主管/ NOC牆](#118-executive--noc-wall-board) | NOC，領導力 | KPI圖磚、可用性公式、依層級的流量、SLO備註 |
| [11.9比例與淨餘空間](#119-ratios--headroom-board-formula-showcase) | 每個人 | 公式Widget展示 |
| [11.10服務總管](#1110-service-explorer-variable-driven) | 每個團隊 | 變數導向、可重複使用 |

**圖表圖例：** **N** =數字，**L** =折線，**A** =區域，**B** =長條，**P** =圓餅圖，**T** =表格。

---

### Golden Signals / SLO板

**目標：**一個畫面會回答「平台目前是否正常？」  — 流量、錯誤、延遲、飽和度。
**時間選擇器：**&#x200B;過去1小時（事件）或24小時（每日檢閱）。

```text
┌──────────── Section: SLO KPIs ────────────────────────────────┐
│ N Requests │ N Error % │ N p95 │ N Apdex │ N Slow % │ N Hosts  │  (6 × 2w×2h)
├──────────── Section: Traffic & Errors ────────────────────────┤
│ A Throughput by service (6w) │ L Error % by service (6w)       │
├──────────── Section: Latency ─────────────────────────────────┤
│ L p50/p95/p99 (6w)           │ L Apdex by service (6w)         │
├──────────── Section: Saturation ──────────────────────────────┤
│ L CPU by host (4w) │ L Memory by host (4w) │ L Load 5m (4w)    │
└───────────────────────────────────────────────────────────────┘
```

**W1 ·要求** — N， 2×2

```sql
SELECT count(*) AS requests FROM Transaction
```

**W2 ·錯誤%** — N，單位`%`，2×2。 臨界值： `0 → Green`、`1 → Amber`、`5 → Red`。

```sql
SELECT percentage(*, WHERE status = 'Error') AS errorRate FROM Transaction
```

**W3 · p95延遲** — N，單位`ms`，2×2。 臨界值： `0 → Green`、`800 → Amber`、`2000 → Red`。

```sql
SELECT percentile(duration, 95) AS p95 FROM Transaction
```

**W4 · Apdex** — N， 2×2。 臨界值： `0 → Red`、`0.7 → Amber`、`0.85 → Green` （最高相符頻帶WINS，因此低分會維持紅色）。

```sql
SELECT apdex(duration, 500) AS apdex FROM Transaction
```

**W5 ·要求> 2 s** — N，單位`%`，2×2。 臨界值： `0 → Green`、`2 → Amber`、`10 → Red`。

```sql
SELECT percentage(*, WHERE duration > 2000) AS slowPct FROM Transaction
```

**W6 ·作用中的主機** — N、2×2

```sql
SELECT uniqueCount(host) AS hosts FROM Transaction
```

**W7 ·服務輸送量** — A， 6×4

```sql
SELECT count(*) AS requests FROM Transaction FACET service TIMESERIES AUTO LIMIT 8
```

**W8 ·服務錯誤%** — L，單位`%`，6×4

```sql
SELECT percentage(*, WHERE status = 'Error') AS errorRate FROM Transaction FACET service TIMESERIES AUTO LIMIT 8
```

**W9 ·延遲百分位數** — L，單位`ms`，6×4

```sql
SELECT percentile(duration, 50) AS p50, percentile(duration, 95) AS p95, percentile(duration, 99) AS p99 FROM Transaction TIMESERIES AUTO
```

**W10 ·依服務排列的Apdex** — L， 6×4

```sql
SELECT apdex(duration, 500) AS apdex FROM Transaction FACET service TIMESERIES AUTO LIMIT 8
```

**W11 ·主機CPU** — L， 4×4

```sql
SELECT average(value) AS cpu FROM CpuUtilization FACET host TIMESERIES AUTO LIMIT 10
```

**W12 ·記憶體（依主機）** — L， 4×4

```sql
SELECT average(value) AS memory FROM MemoryUtilization FACET host TIMESERIES AUTO LIMIT 10
```

**W13 ·載入5m** — L， 4×4

```sql
SELECT average(value) AS load5m FROM LoadAverage5m FACET host TIMESERIES AUTO LIMIT 10
```

---

### 錯誤分類面板

**目標：**&#x200B;從「錯誤率已達」到「此端點、此例外狀況、此追蹤」（按三下）。

```text
Tab "Overview"
│ N Errors │ N Error % │ N Affected traces │ N 5xx count │                 (4 × 3w×2h)
│ L Errors by service (8w) │ P Status code mix (4w)                       │
│ T Errors vs total by service (12w)                                       │
Tab "Drill-down"
│ T 5xx hot spots: service × path × code (12w)                             │
│ T Exception types (6w) │ T Downstream failures (6w)                      │
│ T Latest failing requests – event list (12w×6h)                          │
```

**概觀標籤**

**W1 ·錯誤** — N，臨界值`0 → Green`，`1 → Amber`，`100 → Red`

```sql
SELECT count(*) AS errors FROM Transaction WHERE status = 'Error'
```

**W2 ·錯誤%** — N，單位`%`

```sql
SELECT percentage(*, WHERE status = 'Error') AS errorRate FROM Transaction
```

**W3 ·受影響的追蹤** — N

```sql
SELECT uniqueCount(traceId) AS traces FROM Span WHERE status = 'Error'
```

**W4 · HTTP 5xx** — N，臨界值`0 → Green`，`1 → Red`

```sql
SELECT count(*) AS http5xx FROM Transaction WHERE http.statusCode >= 500
```

**W5 ·服務發生錯誤** — L， 8×4

```sql
SELECT count(*) AS errors FROM Transaction WHERE status = 'Error' FACET service TIMESERIES AUTO LIMIT 10
```

**W6 ·狀態碼組合** — P，4×4

```sql
SELECT count(*) AS requests FROM Transaction WHERE http.statusCode >= 400 FACET http.statusCode LIMIT 10
```

**W7 ·錯誤與服務總數** — T，12×4

```sql
SELECT count(*) AS total, count(*) FILTER (WHERE status = 'Error') AS errors, percentage(*, WHERE status = 'Error') AS errorPct, count(*) FILTER (WHERE http.statusCode >= 500) AS http5xx, count(*) FILTER (WHERE http.statusCode >= 400 AND http.statusCode < 500) AS http4xx FROM Transaction FACET service ORDER BY errors DESC LIMIT 30
```

**向下鑽研索引標籤**

**W8 · 5xx熱點** — T， 12×5

```sql
SELECT count(*) AS errors, percentile(duration, 95) AS p95, latest(traceId) AS lastTraceId FROM Transaction WHERE http.statusCode >= 500 FACET service, urlPath, http.statusCode ORDER BY errors DESC LIMIT 100
```

**W9 ·例外狀況型別** — T， 6×5

```sql
SELECT count(*) AS errors, latest(statusMessage) AS lastMessage FROM Span WHERE status = 'Error' AND error.type != '' FACET service, error.type ORDER BY errors DESC LIMIT 50
```

**W10 ·下游失敗** — T，6×5

```sql
SELECT count(*) AS failures, percentile(duration, 95) AS p95 FROM Span WHERE kind = 'Client' AND status = 'Error' FACET service, peer.service, externalDomain ORDER BY failures DESC LIMIT 50
```

**W11 ·最新的失敗要求** — T，12×6

```sql
SELECT timestamp, service, host, name, urlPath, http.statusCode, duration, statusMessage, traceId FROM Transaction WHERE status = 'Error' LIMIT 200
```

---

### 延遲深度板

**目標：**&#x200B;尋找&#x200B;_位置_&#x200B;時間 — 哪些作業、哪些區段、一致性如何。

```text
│ L p50/p95/p99 (8w)               │ N p99 (4w)                │
│ B Time share by operation (6w)   │ B Slowest ops by p95 (6w) │
│ T Latency profile per operation (12w)                         │
│ T Latency buckets per service (6w) │ B Variability (6w)       │
│ T Transaction composition – SpanBreakdown (12w)               │
│ T Slowest trace per operation (12w)                           │
```

**W1 ·百分位數** — L， `ms`， 8×4

```sql
SELECT percentile(duration, 50) AS p50, percentile(duration, 95) AS p95, percentile(duration, 99) AS p99 FROM Transaction TIMESERIES AUTO
```

**W2 · p99** — N， `ms`，臨界值`0 → Green`， `1500 → Amber`， `4000 → Red`， 4×4

```sql
SELECT percentile(duration, 99) AS p99 FROM Transaction
```

**W3 ·作業的時間共用** — B， `%`， 6×4

```sql
SELECT percentOfTotal(duration) AS timeShare FROM Transaction FACET name LIMIT 15
```

**W4 ·最慢的作業(p95)** — B， `ms`， 6×4

```sql
SELECT percentile(duration, 95) AS p95 FROM Transaction FACET name ORDER BY p95 DESC LIMIT 15
```

**W5 ·延遲設定檔** — T，12×5

```sql
SELECT count(*) AS calls, average(duration) AS avgMs, percentile(duration, 50) AS p50, percentile(duration, 95) AS p95, percentile(duration, 99) AS p99, max(duration) AS maxMs FROM Transaction FACET service, name ORDER BY p95 DESC LIMIT 100
```

**W6 ·延遲值區** — T，6×4

```sql
SELECT count(*) FILTER (WHERE duration <= 100) AS fast, count(*) FILTER (WHERE duration > 100 AND duration <= 500) AS ok, count(*) FILTER (WHERE duration > 500 AND duration <= 2000) AS slow, count(*) FILTER (WHERE duration > 2000) AS verySlow FROM Transaction FACET service LIMIT 20
```

**W7 ·延遲變動性** — B， `ms`， 6×4

```sql
SELECT stddev(duration) AS stddevMs FROM Transaction FACET service ORDER BY stddevMs DESC LIMIT 15
```

**W8 ·交易組合** — T，單位`ms`，12×5

```sql
SELECT sum(callCount) AS calls, average(avgDurationMs) AS avgMs FROM SpanBreakdown FACET service, rootOperation, segment ORDER BY calls DESC LIMIT 100
```

**W9 ·每個作業最慢的追蹤** — T，12×5

```sql
SELECT count(*) AS calls, max(duration) AS slowestMs, latest(traceId, duration) AS slowestTraceId, latest(host, duration) AS slowestHost FROM Transaction FACET name ORDER BY slowestMs DESC LIMIT 50
```

---

### 下游相依性面板

**目標：**&#x200B;是我們還是他們？ 我們呼叫的每個資料庫、API、佇列和RPC的健康狀況。

```text
│ N Outbound calls │ N Outbound error % │ N Outbound p95 │            (3 × 4w×2h)
│ T Downstream services (12w)                                       │
│ L External domain p95 (6w) │ T External domains (6w)              │
│ B DB p95 by system/op (6w) │ T Slow DB namespaces (6w)            │
│ L Messages per destination (6w) │ T Producer vs consumer (6w)     │
│ T RPC methods (12w)                                               │
```

**W1 ·傳出呼叫** — N

```sql
SELECT count(*) AS calls FROM Span WHERE kind = 'Client'
```

**W2 ·傳出錯誤%** — N， `%`，臨界值`0 → Green`， `2 → Amber`， `10 → Red`

```sql
SELECT percentage(*, WHERE status = 'Error') AS errorPct FROM Span WHERE kind = 'Client'
```

**W3 ·輸出p95** — N， `ms`

```sql
SELECT percentile(duration, 95) AS p95 FROM Span WHERE kind = 'Client'
```

**W4 ·下游服務** — T，12×5

```sql
SELECT count(*) AS calls, percentile(duration, 95) AS p95, percentage(*, WHERE status = 'Error') AS errorPct FROM Span WHERE kind = 'Client' FACET service, peer.service ORDER BY calls DESC LIMIT 100
```

**W5 ·外部網域p95** — L， `ms`， 6×4

```sql
SELECT percentile(duration, 95) AS p95 FROM Span WHERE kind = 'Client' AND externalDomain != '' FACET externalDomain TIMESERIES AUTO LIMIT 8
```

**W6 ·外部網域** — T，6×4

```sql
SELECT count(*) AS calls, percentile(duration, 95) AS p95, count(*) FILTER (WHERE status = 'Error') AS errors FROM Span WHERE kind = 'Client' AND externalDomain != '' FACET externalDomain ORDER BY calls DESC LIMIT 50
```

**W7 · DB p95** — B， `ms`， 6×4

```sql
SELECT percentile(duration, 95) AS p95 FROM Span WHERE db.system != '' FACET db.system, db.operation ORDER BY p95 DESC LIMIT 20
```

**W8 ·慢速DB名稱空間** — T，6×4

```sql
SELECT count(*) AS calls, average(duration) AS avgMs, percentile(duration, 99) AS p99, count(*) FILTER (WHERE status = 'Error') AS errors FROM Span WHERE db.system != '' FACET service, db.system, db.namespace ORDER BY p99 DESC LIMIT 50
```

**W9 ·每個目的地的訊息** — L， 6×4

```sql
SELECT count(*) AS messages FROM Span WHERE messaging.system != '' FACET messaging.destination TIMESERIES AUTO LIMIT 10
```

**W10 ·製作者vs消費者** — T，6×4

```sql
SELECT count(*) FILTER (WHERE kind = 'Producer') AS produced, count(*) FILTER (WHERE kind = 'Consumer') AS consumed, percentile(duration, 95) AS p95 FROM Span WHERE messaging.system != '' FACET messaging.system, messaging.destination LIMIT 50
```

**W11 · RPC方法** — T， 12×4

```sql
SELECT count(*) AS calls, percentile(duration, 95) AS p95, percentage(*, WHERE status = 'Error') AS errorPct FROM Span WHERE rpc.system != '' FACET service, rpc.system, rpc.method ORDER BY calls DESC LIMIT 50
```

---

### AEM作者/發佈/Dispatcher展示板

**目標：**依層級分組的AEM操作展示板，映象傳統New Relic AEM配置。
使用名為`PUBLISH`、`AUTHOR`、`DISPATCHER`的&#x200B;**區段標題**&#x200B;以及結尾包含`- PUBLISH` / `- AUTHOR`的Widget標題來取得色彩提示。

```text
Tab "Tiers"
├─ Section: OVERVIEW
│ A Traffic by tier (8w) │ T Content mix by tier (4w)
├─ Section: PUBLISH
│ N Requests - PUBLISH │ N Error % - PUBLISH │ N p95 - PUBLISH │ N Apdex - PUBLISH
│ L p95 by host - PUBLISH (6w) │ L CPU - PUBLISH (6w)
│ L Heap - PUBLISH (6w) │ T Slowest /bin servlets - PUBLISH (6w)
├─ Section: AUTHOR
│ N Requests - AUTHOR │ N Error % - AUTHOR │ N p95 - AUTHOR │ N Active authors' hosts - AUTHOR
│ L p95 by host - AUTHOR (6w) │ L Heap - AUTHOR (6w)
├─ Section: DISPATCHER
│ L Dispatcher req/s (8w) │ L CPU - DISPATCHER (4w)
Tab "Code hot spots"
│ T Slow Sling scripts (6w) │ T Slow JSPs (6w)
│ T Hottest JCR paths (6w)  │ B JCR operation mix (6w)
│ T Sling request URIs (6w) │ T GraphQL operations (6w)
│ B Sling servlet time share (12w)
```

**概觀**

**W1 ·依階層的流量** — A， 8×4

```sql
SELECT count(*) AS requests FROM Transaction WHERE hostTierPrefix != '' FACET hostTierPrefix TIMESERIES AUTO LIMIT 10
```

**W2 ·依階層的內容組合** — T，4×4

```sql
SELECT count(*) FILTER (WHERE urlPath LIKE '/content/%') AS pages, count(*) FILTER (WHERE urlPath LIKE '/bin/%') AS servlets, count(*) FILTER (WHERE urlPath LIKE '/content/dam/%') AS assets FROM Transaction FACET hostTierPrefix LIMIT 20
```

**發佈區段**

**W3 ·要求 — 發佈** — N

```sql
SELECT count(*) AS requests FROM Transaction WHERE host LIKE '%publish%'
```

**W4 ·錯誤% — 發佈** — N， `%`，臨界值`0 → Green`，`1 → Amber`，`5 → Red`

```sql
SELECT percentage(*, WHERE status = 'Error') AS errorRate FROM Transaction WHERE host LIKE '%publish%'
```

**W5 · p95 — 發佈** — N， `ms`，臨界值`0 → Green`，`1000 → Amber`，`3000 → Red`

```sql
SELECT percentile(duration, 95) AS p95 FROM Transaction WHERE host LIKE '%publish%'
```

**W6 · Apdex - PUBLISH** — N，臨界值`0 → Red`，`0.7 → Amber`，`0.85 → Green`

```sql
SELECT apdex(duration, 500) AS apdex FROM Transaction WHERE host LIKE '%publish%'
```

**W7 · p95依主機 — PUBLISH** — L， `ms`， 6×4

```sql
SELECT percentile(duration, 95) AS p95 FROM Transaction WHERE host LIKE '%publish%' FACET host TIMESERIES AUTO LIMIT 10
```

**W8 · CPU — 發佈** — L， `%`， 6×4

```sql
SELECT average(value) AS cpu FROM CpuUtilization WHERE host LIKE '%publish%' FACET host TIMESERIES AUTO
```

**W9 ·棧積 — PUBLISH** — L， `MB`， 6×4

```sql
SELECT average(value) AS heapMb FROM JvmHeapUsed WHERE host LIKE '%publish%' FACET host TIMESERIES AUTO
```

**W10 ·最慢的/bin servlet - PUBLISH** — T，6×4

```sql
SELECT count(*) AS calls, percentile(duration, 95) AS p95, percentage(*, WHERE status = 'Error') AS errorPct FROM Transaction WHERE host LIKE '%publish%' AND urlPath LIKE '/bin/%' FACET urlPath ORDER BY p95 DESC LIMIT 30
```

**作者區段** — 重複W3-W9並將`'%publish%'`變更為`'%author%'`，並將標題字尾變更為`- AUTHOR`。 額外Widget：

**W11 ·使用中的作者主機 — AUTHOR** — N

```sql
SELECT uniqueCount(host) AS hosts FROM Transaction WHERE host LIKE '%author%'
```

**Dispatcher區段**

**W12 · Dispatcher req/s** — L， `req/s`， 8×4

```sql
SELECT rate(value) AS requestsPerSecond FROM DispatcherRequestRate FACET host TIMESERIES AUTO
```

**W13 · CPU - DISPATCHER** — L， `%`， 4×4

```sql
SELECT average(value) AS cpu FROM CpuUtilization WHERE host LIKE '%dispatcher%' FACET host TIMESERIES AUTO
```

**程式碼熱點索引標籤**

**W14 ·慢速Sling指令碼** — T

```sql
SELECT count(*) AS executions, percentile(duration, 95) AS p95, sum(duration) AS totalMs FROM Span WHERE sling.script != '' FACET sling.script ORDER BY totalMs DESC LIMIT 50
```

**W15 ·慢速JSP** — T

```sql
SELECT count(*) AS executions, percentile(duration, 95) AS p95 FROM Span WHERE jsp.path != '' FACET jsp.path ORDER BY p95 DESC LIMIT 50
```

**W16 ·最熱的JCR路徑** — T

```sql
SELECT count(*) AS operations, percentile(duration, 95) AS p95, sum(duration) AS totalMs FROM Span WHERE jcr.path != '' FACET jcr.operation, jcr.path ORDER BY totalMs DESC LIMIT 50
```

**W17 · JCR作業組合** — B

```sql
SELECT count(*) AS operations FROM Span WHERE jcr.operation != '' FACET jcr.operation LIMIT 20
```

**W18 · Sling要求URI** — T

```sql
SELECT count(*) AS requests, percentile(duration, 95) AS p95 FROM Transaction WHERE sling.requestUri != '' FACET sling.requestUri ORDER BY requests DESC LIMIT 100
```

**W19 · GraphQL作業** — T

```sql
SELECT count(*) AS calls, percentile(duration, 95) AS p95, count(*) FILTER (WHERE status = 'Error') AS errors FROM Span WHERE graphql.operation != '' FACET graphql.operation ORDER BY calls DESC LIMIT 50
```

**W20 · Sling servlet時間共用** — B， `%`， 12×4

```sql
SELECT percentOfTotal(duration) AS timeShare FROM Span WHERE sling.servlet != '' FACET sling.servlet LIMIT 20
```

---

### 基礎架構容量板

**目標：**跨主機的容量規劃和飽和度捕捉。
**時間選擇器：** 7天用於趨勢（統計可讓此速度更快），1小時用於事件。

```text
Tab "Compute"
│ N Hosts reporting │ N Peak CPU % │ N Peak memory % │ N Max load 5m │
│ L CPU by host (6w) │ L Memory by host (6w)                         │
│ L CPU user / system / iowait / steal (4 × 3w)                     │
│ T Host snapshot (12w)                                              │
│ T Top processes by CPU (6w) │ T Top processes by memory (6w)       │
Tab "Storage & Network"
│ T Disk used per mount (6w) │ N Lowest free root disk (6w)          │
│ L Disk read (6w) │ L Disk write (6w)                               │
│ L Network rx (6w) │ L Network tx (6w)                              │
│ L Total network I/O for one host – formula (12w)                   │
```

**W1 ·主機報告** — N

```sql
SELECT uniqueCount(host) AS hosts FROM CpuUtilization
```

**W2 ·尖峰CPU** — N， `%`，臨界值`0 → Green`，`75 → Amber`，`90 → Red`

```sql
SELECT max(value) AS peakCpu FROM CpuUtilization
```

**W3 ·尖峰記憶體** — N， `%`，臨界值`0 → Green`，`80 → Amber`，`92 → Red`

```sql
SELECT max(value) AS peakMemory FROM MemoryUtilization
```

**W4 ·最大負載5m** — N

```sql
SELECT max(value) AS maxLoad FROM LoadAverage5m
```

**W5 ·主機CPU** — L， `%`， 6×4

```sql
SELECT average(value) AS cpu FROM CpuUtilization FACET host TIMESERIES AUTO
```

**W6 ·記憶體（依主機）** — L， `%`， 6×4

```sql
SELECT average(value) AS memory FROM MemoryUtilization FACET host TIMESERIES AUTO
```

**W7-W10 ·依模式CPU** — L， `%`，各3×4

```sql
SELECT average(value) AS userPct FROM CpuUserPercent FACET host TIMESERIES AUTO LIMIT 10
```

```sql
SELECT average(value) AS systemPct FROM CpuSystemPercent FACET host TIMESERIES AUTO LIMIT 10
```

```sql
SELECT average(value) AS iowaitPct FROM CpuIowaitPercent FACET host TIMESERIES AUTO LIMIT 10
```

```sql
SELECT average(value) AS stealPct FROM CpuStealPercent FACET host TIMESERIES AUTO LIMIT 10
```

**W11 ·主機快照** — T，12×5

```sql
SELECT average(value) AS avgCpu, percentile(value, 95) AS p95Cpu, max(value) AS peakCpu, latest(value) AS nowCpu FROM CpuUtilization FACET host ORDER BY p95Cpu DESC LIMIT 100
```

**W12 · CPU排名最前的程式** — T，6×5

```sql
SELECT average(value) AS cpuPct, max(value) AS peakPct FROM ProcessCpuUtilization FACET host, process.name ORDER BY cpuPct DESC LIMIT 30
```

**W13 ·記憶體排名最前的處理序** — T， `MB`， 6×5

```sql
SELECT average(value) AS memMb, max(value) AS peakMb FROM ProcessMemoryUsage FACET host, process.name ORDER BY memMb DESC LIMIT 30
```

**儲存和網路索引標籤**

**W14 ·每個掛載使用的磁碟** — T， `%`

```sql
SELECT latest(value) AS usedPct, max(value) AS peakPct FROM DiskUtilization FACET host, mountpoint ORDER BY usedPct DESC LIMIT 50
```

**W15 ·最低可用根磁碟** — N，單位`bytes`，臨界值`0 → Red`，`5000000000 → Amber`，`20000000000 → Green`

```sql
SELECT min(value) AS freeBytes FROM DiskFreeBytes
```

**W16 / W17 ·磁碟輸送量** — L， `bytes/s`

```sql
SELECT rate(value) AS readBytesPerSec FROM DiskReadRate FACET host TIMESERIES AUTO
```

```sql
SELECT rate(value) AS writeBytesPerSec FROM DiskWriteRate FACET host TIMESERIES AUTO
```

**W18 / W19 ·網路輸送量** — L， `bytes/s`

```sql
SELECT rate(value) AS rxBytesPerSec FROM NetworkReceiveRate FACET host TIMESERIES AUTO
```

```sql
SELECT rate(value) AS txBytesPerSec FROM NetworkTransmitRate FACET host TIMESERIES AUTO
```

**W20 ·單一主機的網路I/O總數（公式）** — 公式Widget， L， `bytes/s`， 12×4。 公式不能`FACET`，且`rate()`的範圍必須限定至一個主機（請參閱[已組織的量度實體](#53-curated-metric-entities)），因此請釘選主機 — 將`web-publish-01`取代為W5中的實際主機名稱。

查詢`netRx`：

```sql
SELECT rate(value) AS v FROM NetworkReceiveRate WHERE host = 'web-publish-01' TIMESERIES AUTO
```

查詢`netTx`：

```sql
SELECT rate(value) AS v FROM NetworkTransmitRate WHERE host = 'web-publish-01' TIMESERIES AUTO
```

公式：

```text
netRx + netTx
```

---

### JVM健康情況展示板

**目標：**&#x200B;在棧積壓力、執行緒洩漏和JVM CPU尖峰變成中斷之前攔截它們。

```text
│ N Heap used % (formula) │ N Peak threads │ N JVM CPU % │ N Max heap MB │
│ L Heap used % over time – formula (12w)                                │
│ L Heap used by host (6w) │ T Heap limit by host (6w)                    │
│ L Threads by host (6w)   │ L JVM CPU by host (6w)                       │
│ T Memory pools (12w)                                                   │
```

**W1 ·棧積已使用% （公式）** — 公式， N， `%`，臨界值`0 → Green`， `75 → Amber`， `90 → Red`

查詢`used`：

```sql
SELECT average(value) AS v FROM JvmHeapUsed
```

查詢`limit`：

```sql
SELECT average(value) AS v FROM JvmHeapLimit
```

公式：

```text
(used / limit) * 100
```

**W2 ·尖峰執行緒** — N

```sql
SELECT max(value) AS peakThreads FROM JvmThreadCount
```

**W3 · JVM CPU** — N， `%`

```sql
SELECT average(value) AS jvmCpu FROM JvmCpuRecentUtilization
```

**W4 ·已使用棧積上限** — N， `MB`

```sql
SELECT max(value) AS maxHeapMb FROM JvmHeapUsed
```

**W5 ·棧積已使用%一段時間（公式）** — 公式， L， `%`， 12×4

查詢`used`：

```sql
SELECT average(value) AS v FROM JvmHeapUsed TIMESERIES AUTO
```

查詢`limit`：

```sql
SELECT average(value) AS v FROM JvmHeapLimit TIMESERIES AUTO
```

公式：

```text
(used / limit) * 100
```

> [!NOTE]
> 若要將公式範圍設定為單一主機，請新增`WHERE host = 'my-publish-01'`至&#x200B;**both**&#x200B;個查詢。

**W6 ·主機使用的棧積** — L， `MB`， 6×4

```sql
SELECT average(value) AS heapMb FROM JvmHeapUsed FACET host TIMESERIES AUTO
```

**W7 ·依主機的棧積限制** — T， `MB`， 6×4

```sql
SELECT latest(value) AS heapLimitMb FROM JvmHeapLimit FACET host LIMIT 50
```

**W8 ·主機Threads** — L， 6×4

```sql
SELECT latest(value) AS threads FROM JvmThreadCount FACET host TIMESERIES AUTO
```

**W9 · JVM CPU，依主機** — L， `%`， 6×4

```sql
SELECT average(value) AS jvmCpu FROM JvmCpuRecentUtilization FACET host TIMESERIES AUTO
```

**W10 ·記憶體集區** — T，12×5

```sql
SELECT latest(value) AS bytesUsed, max(value) AS peakBytes FROM MetricSum WHERE metric = 'jvm.memory.used' FACET host, pool ORDER BY peakBytes DESC LIMIT 100
```

---

### 11.8高階主管/NOC牆上型主機板

**目標：**&#x200B;可瀏覽、有色彩編碼的牆面熒幕。 大多數Widget具有臨界值，每列一個趨勢。

```text
│ N Requests │ N Availability % (formula) │ N Apdex │ N p95 │ N 5xx │ N Hosts │
│ L Throughput (12w)                                                          │
│ B Requests by service (6w) │ P Traffic by tier (6w)                         │
│ Text: "SLO: 99.9 % availability, p95 < 800 ms. Owner: platform-sre" (12w×1h) │
```

**W1 ·請求** — N

```sql
SELECT count(*) AS requests FROM Transaction
```

**W2 ·使用狀態% （公式）** — 公式， N， `%`，臨界值`0 → Red`， `99 → Amber`， `99.9 → Green`

查詢`ok`：

```sql
SELECT count(*) AS v FROM Transaction WHERE status != 'Error'
```

查詢`total`：

```sql
SELECT count(*) AS v FROM Transaction
```

公式：

```text
(ok / total) * 100
```

**W3 · Apdex** — N，臨界值`0 → Red`，`0.7 → Amber`，`0.85 → Green`

```sql
SELECT apdex(duration, 500) AS apdex FROM Transaction
```

**W4 · p95** — N， `ms`，臨界值`0 → Green`，`800 → Amber`，`2000 → Red`

```sql
SELECT percentile(duration, 95) AS p95 FROM Transaction
```

**W5 · 5xx** — N，臨界值`0 → Green`，`1 → Amber`，`50 → Red`

```sql
SELECT count(*) AS http5xx FROM Transaction WHERE http.statusCode >= 500
```

**W6 ·主機** — N

```sql
SELECT uniqueCount(host) AS hosts FROM Transaction
```

**W7 ·輸送量** — L， 12×4

```sql
SELECT count(*) AS requests FROM Transaction TIMESERIES AUTO
```

**W8 ·服務要求** — B，6×4

```sql
SELECT count(*) AS requests FROM Transaction FACET service LIMIT 10
```

**W9 ·依階層的流量** — P，6×4

```sql
SELECT count(*) AS requests FROM Transaction WHERE hostTierPrefix != '' FACET hostTierPrefix LIMIT 10
```

**W10 · SLO備註** — 文字Widget，12×1。

---

### 比例與預留空間展示板（公式展示）

**目標：**&#x200B;需要多個查詢的衍生KPI。

**W1 ·每個主機的要求** — 公式， L， `req/host`

查詢`requests`：

```sql
SELECT count(*) AS v FROM Transaction TIMESERIES AUTO
```

查詢`hosts`：

```sql
SELECT uniqueCount(host) AS v FROM Transaction TIMESERIES AUTO
```

公式：

```text
requests / hosts
```

**W2 · 5xx共用所有錯誤** — 公式， N， `%`

查詢`e5xx`：

```sql
SELECT count(*) AS v FROM Transaction WHERE http.statusCode >= 500
```

查詢`errors`：

```sql
SELECT count(*) AS v FROM Transaction WHERE http.statusCode >= 400
```

公式：

```text
(e5xx / errors) * 100
```

**W3 ·資料庫中的時間與伺服器總時間** — 公式，L， `%`

查詢`db`：

```sql
SELECT sum(duration) AS v FROM Span WHERE db.system != '' TIMESERIES AUTO
```

查詢`server`：

```sql
SELECT sum(duration) AS v FROM Transaction TIMESERIES AUTO
```

公式：

```text
(db / server) * 100
```

**W4 ·棧積空間(MB)** — 公式， L， `MB`

查詢`limit`：

```sql
SELECT average(value) AS v FROM JvmHeapLimit TIMESERIES AUTO
```

查詢`used`：

```sql
SELECT average(value) AS v FROM JvmHeapUsed TIMESERIES AUTO
```

公式：

```text
limit - used
```

**W5 ·單一主機的網路輸出/輸入比例** — 公式，L （釘選主機；請參閱[已組織的量度實體](#53-curated-metric-entities)）

查詢`rx`：

```sql
SELECT rate(value) AS v FROM NetworkReceiveRate WHERE host = 'web-publish-01' TIMESERIES AUTO
```

查詢`tx`：

```sql
SELECT rate(value) AS v FROM NetworkTransmitRate WHERE host = 'web-publish-01' TIMESERIES AUTO
```

公式：

```text
tx / rx
```

**W6 ·發佈與作者流量比率** — 公式，N

查詢`publish`：

```sql
SELECT count(*) AS v FROM Transaction WHERE host LIKE '%publish%'
```

查詢`author`：

```sql
SELECT count(*) AS v FROM Transaction WHERE host LIKE '%author%'
```

公式：

```text
publish / author
```

**W7 ·加權錯誤預算加深(SLO 99.9 %)** — 公式，L

查詢`errors`：

```sql
SELECT count(*) AS v FROM Transaction WHERE status = 'Error' TIMESERIES AUTO
```

查詢`total`：

```sql
SELECT count(*) AS v FROM Transaction TIMESERIES AUTO
```

公式（燒錄率：1.0 =完全依照排程沖銷預算）：

```text
(errors / total) / 0.001
```

---

### 服務總管（變數驅動）

**目標：**&#x200B;任何團隊都可以指向&#x200B;_其_&#x200B;服務和環境的一個展示板。

> [!WARNING]
> 請參閱[教學課程3步驟8.7](#87-make-dashboards-reusable-with-variables)中的&#x200B;**目前組建限制**：尚未預覽或儲存Widget SQL中的`{{variables}}`。 在修正之前，請以常值取代每個`{{…}}` （例如`'checkout'`），並使用&#x200B;_複製_&#x200B;來複製每個服務的展示板。 當支援端對端替代時，以下的設計已準備就緒。

**變數**

| 名稱 | 標籤 | 類型 | 多 | 數值 | 定義 |
| -------- | ------------------- | ------------ | ----- | ------- | ---------------------------------------------------------- |
| `svc` | 服務 | 從查詢 | ✔ |         | `SELECT count(*) FROM Transaction FACET service LIMIT 200` |
| `env` | 環境 | 從查詢 | ✔ |         | `SELECT count(*) FROM Span FACET environment LIMIT 50` |
| `slowMs` | 慢速臨界值（毫秒） | 文字方塊 |       | ✔ | 預設`2000` |

```text
Section: "{{svc}} in {{env}}"
│ N Requests │ N Error % │ N p95 │ N Slow requests │
│ L Throughput by service (6w) │ L p95 by service (6w)   │
│ T Operations (12w)                                     │
│ T Slow requests over {{slowMs}} ms – events (12w)       │
│ T Downstream calls (12w)                               │
```

**W1 ·請求 —{{svc}}** — N

```sql
SELECT count(*) AS requests FROM Transaction WHERE service IN ({{svc}}) AND environment IN ({{env}})
```

**W2 ·錯誤%** — N， `%`

```sql
SELECT percentage(*, WHERE status = 'Error') AS errorRate FROM Transaction WHERE service IN ({{svc}}) AND environment IN ({{env}})
```

**W3 · p95** — N， `ms`

```sql
SELECT percentile(duration, 95) AS p95 FROM Transaction WHERE service IN ({{svc}}) AND environment IN ({{env}})
```

**W4 ·緩慢要求** — N

```sql
SELECT count(*) AS slow FROM Transaction WHERE service IN ({{svc}}) AND environment IN ({{env}}) AND duration > {{slowMs}}
```

**W5 ·服務輸送量** — L

```sql
SELECT count(*) AS requests FROM Transaction WHERE service IN ({{svc}}) AND environment IN ({{env}}) FACET service TIMESERIES AUTO
```

**W6 · p95 （依服務）** — L， `ms`

```sql
SELECT percentile(duration, 95) AS p95 FROM Transaction WHERE service IN ({{svc}}) AND environment IN ({{env}}) FACET service TIMESERIES AUTO
```

**W7 ·作業** — T

```sql
SELECT count(*) AS calls, percentile(duration, 95) AS p95, percentage(*, WHERE status = 'Error') AS errorPct FROM Transaction WHERE service IN ({{svc}}) AND environment IN ({{env}}) FACET service, name ORDER BY calls DESC LIMIT 100
```

**W8 ·緩慢要求（事件）** — T

```sql
SELECT timestamp, service, host, name, urlPath, duration, traceId FROM Transaction WHERE service IN ({{svc}}) AND duration > {{slowMs}} ORDER BY duration DESC LIMIT 200
```

**W9 ·下游呼叫** — T

```sql
SELECT count(*) AS calls, percentile(duration, 95) AS p95, count(*) FILTER (WHERE status = 'Error') AS errors FROM Span WHERE kind = 'Client' AND service IN ({{svc}}) FACET peer.service, externalDomain, db.system ORDER BY calls DESC LIMIT 50
```
