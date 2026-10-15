---
title: SQL參考
description: 參考Observability Insights支援的SQL語法、子句和函式。
source-git-commit: 91a61d33010192cedce7d40a45ffe86f32e4824b
workflow-type: tm+mt
source-wordcount: '1505'
ht-degree: 11%
---

## SQL參考 {#sql-reference}

此參考資料涵蓋SQL語法、子句、函式和運運算元。 展開每個專案以取得詳細資訊和範例。

### 查詢元件

```text
SELECT <select-item> [, <select-item> ...]
FROM   <Entity>
[WHERE      <condition>]
[FACET      <attribute> [, <attribute> ...]]          -- up to 5
[SINCE      <n> <unit> [AGO]]
[UNTIL      NOW | <n> <unit> [AGO]]
[TIMESERIES AUTO | MAX | <n> <unit>]
[ORDER BY   <alias-or-selected-name> [ASC | DESC]]
[LIMIT      <1..5000>]

<select-item> := <function>(<args>) [FILTER (WHERE <condition>)] [AS <alias>]
               | <attribute> [AS <alias>]
<unit>        := second | seconds | minute | minutes | hour | hours | day | days | week | weeks
```

> **附註**
>
> 上述`--`註解僅為檔案。 SQL本身不接受註解。

### 必要條款

#### `SELECT` — 要傳回的專案

+++檢視詳細資料

`SELECT`列出&#x200B;**原始屬性** （事件查詢）或&#x200B;**彙總函式** （彙總查詢）。 您不能將兩者混合使用。

| 表單 | 範例 | 結果 |
| ------------ | -------------------------------------------------- | ------------------------------------------------- |
| 屬性 | `SELECT timestamp, name, duration` | 個別事件的列（預設為100） |
| 彙總 | `SELECT count(*), average(duration)` | 每個彙總一個值（每個面向/時段） |
| 含別名 | `SELECT count(*) AS requests` | 資料行標籤`requests` |
| 引號別名 | `SELECT percentile(duration, 95) AS 'p95 latency'` | 帶有空格的標籤 |

事件清單：

```sql
SELECT timestamp, service, name, duration, status, traceId FROM Transaction WHERE status = 'Error' LIMIT 50
```

數個彙總：

```sql
SELECT count(*) AS requests, average(duration) AS avgMs, max(duration) AS maxMs FROM Transaction
```

規則：

- 不支援`SELECT *`。 列出您想要的屬性。
- 不支援屬性或彙總(`sum(a) / count(*)`)之間的數學運算。 使用[公式小工具](#86-combine-queries-with-formula-widgets)。
- 混合屬性和彙總失敗，因為&#x200B;_原始屬性不能與彙總混合；請使用FACET將結果分組_。

+++

#### `FROM` — 要查詢的實體

+++檢視詳細資料

`FROM`名稱與[資料字典](#data-dictionary)中的&#x200B;**一個**&#x200B;實體完全相同。 實體名稱不區分大小寫。

```sql
SELECT count(*) FROM Transaction
```

```sql
FROM Span SELECT count(*) FACET kind
```

每個實體都有固定的範圍。 例如，`Transaction`永遠是伺服器範圍，而`CpuUtilization`永遠是CPU忙碌量度，並將`value`轉換為百分比。 您無法在單一查詢中查詢多個實體。 若要合併實體，請使用[公式Widget](#86-combine-queries-with-formula-widgets)。

+++

### 選擇性子句

#### `AS` — 為欄加上標籤

+++檢視詳細資料

使用`AS`為彙總或屬性提供可讀的名稱。 別名會變成欄標題、圖例標籤以及您在`ORDER BY`中參照的名稱。

```sql
SELECT count(*) AS requests, percentile(duration, 95) AS p95 FROM Transaction FACET service ORDER BY p95 DESC LIMIT 10
```

請使用單引號字串做為帶有空格的標籤：

```sql
SELECT percentile(duration, 95) AS 'p95 latency (ms)' FROM Transaction TIMESERIES AUTO
```

+++

#### `FACET` — 群組結果

+++檢視詳細資料

`FACET`依據最多&#x200B;**5**&#x200B;個屬性（例如SQL `GROUP BY`）將結果分組。

```sql
SELECT count(*) FROM Transaction FACET service, http.method, http.statusCode LIMIT 30
```

| 查詢 | `LIMIT n`的意義 | 預設 |
| ---------------------------- | ----------------------------------------------------------------------------------------------------------------------------- | --------------------------------- |
| 不含`TIMESERIES`的`FACET` | 列數 | 10 |
| `FACET`與`TIMESERIES` | **前n個系列**，依整個視窗中第一個彙總的總和排名。 系統會為這些序列保留每個時段。 | 所有系列（上限為5,000列） |

多面向時間序列，前5名：

```sql
SELECT count(*) AS requests FROM Transaction FACET service TIMESERIES AUTO LIMIT 5
```

Facet值可以是字串或數字。 空白字串形成自己的群組。 使用`WHERE attr != ''`排除這些專案。

+++

#### `FILTER` — 條件彙總

+++檢視詳細資料

將`FILTER (WHERE condition)`附加至任何彙總，以計算資料列子集上的彙總。 這可讓您一次比較多個條件。

```sql
SELECT count(*) AS total, count(*) FILTER (WHERE status = 'Error') AS errors, percentile(duration, 95) FILTER (WHERE http.method = 'GET') AS getP95 FROM Transaction FACET service LIMIT 20
```

`FILTER`不允許搭配`apdex`、`rate`或`derivative`使用。 `percentage`已採用自己的條件。

+++

#### `LIMIT` — 多少列或數列

+++檢視詳細資料

`LIMIT`採用正整數。 超過5,000的值會固定為5,000。

| 結果形狀 | 預設 |
| ------------ | ---------- |
| 告示板 | 1 |
| Facet | 10 |
| 時間序列 | 5,000列 |
| 事件 | 100 |

```sql
SELECT count(*) FROM Transaction FACET urlPath LIMIT 50
```

>[!TIP]
>
> 一律在查詢導向變數和高基數Facet (`urlPath`， `traceId`， `jcr.path`， `url`)上設定`LIMIT`。

+++

#### `ORDER BY` — 排序結果

+++檢視詳細資料

`ORDER BY`依選取的&#x200B;**別名或選取的屬性**&#x200B;排序。 預設方向為`DESC`。

```sql
SELECT count(*) AS requests, percentile(duration, 95) AS p95 FROM Transaction FACET name ORDER BY p95 DESC LIMIT 20
```

| 結果形狀 | 預設順序 |
| ----------------- | --------------------------------------------------------------------------- |
| 告示板/面向 | 第一個彙總，`DESC` |
| 時間序列 | 時間，`ASC` |
| 事件 | 第一個選取的資料行`DESC` （因此將`timestamp`放在「最新優先」的第一個資料行） |

規則：

- 參照未選取的專案失敗，因為&#x200B;_ORDER BY &#39;x&#39;必須參照選取的值或別名_。
- 請勿將`ORDER BY`新增至`TIMESERIES`查詢。 它會覆寫圖表所需的時間順序。

+++

#### `SINCE` — 視窗的開始

+++檢視詳細資料

`SINCE n unit [AGO]`設定回溯期間的開始時間。 單位： `second`、`minute`、`hour`、`day`、`week` （單數或複數）。 `AGO`是選用專案。

```sql
SELECT count(*) FROM Transaction SINCE 6 hours ago
```

- 預設為 `SINCE 1 hour ago`。
- 時間範圍不能超過&#x200B;**31天** （_時間範圍不能超過31天_）。
- 不支援絕對日期(`SINCE '2026-01-01'`)和`TODAY` / `YESTERDAY`。

>[!IMPORTANT]
>
> 在控制面板和Explore中，**時間選擇器會覆寫** `SINCE`和`UNTIL`。 請參閱[時間視窗優先順序](#time-window-precedence)。

+++

#### `TIMESERIES` — 繪製一段時間內

+++檢視詳細資料

`TIMESERIES`會針對每個時段（每個Facet值）傳回一列，為折線、區域或長條圖做好準備。

| 表單 | 貯體大小 |
| ---------------------- | --------------------------------------------------------------------------------------------------------------------------------------------- |
| `TIMESERIES 5 minutes` | 完全符合您的要求（任何單位） |
| `TIMESERIES AUTO` | 視窗÷120左右，四捨五入為1秒/ 5秒/ 10秒/ 15秒/ 30秒/ 1米/ 5米/ 10米/ 30米/ 1小時/ 3小時/ 6小時/ 12小時/ 1天。 **永遠不超過2分鐘。** |
| `TIMESERIES MAX` | 1秒貯體。 僅供幾分鐘使用。 |

一般`AUTO`貯體：

| 視窗 | 1小時 | 6小時 | 24小時 | 7天 | 30天 |
| ------ | ----- | ----- | ------ | --- | ---- |
| 貯體 | 2分鐘 | 5分鐘 | 15分鐘 | 3小時 | 6小時 |

```sql
SELECT percentile(duration, 95) AS p95 FROM Transaction TIMESERIES AUTO
```

```sql
SELECT count(*) FROM Transaction TIMESERIES 1 hour SINCE 1 day ago
```

>[!TIP]
>
> 偏好儀表板上的`AUTO`。 不管時間選擇器怎麼說，它都會保留約120點。 這樣還可以避免每個主機的「損毀」行數，因為主機量度大約每90秒就會刮掉一次。

+++

#### `UNTIL` — 視窗結尾

+++檢視詳細資料

`UNTIL NOW` （預設）或`UNTIL n unit [AGO]`。 將其與`SINCE`結合，檢視過去的視窗：

```sql
SELECT count(*) AS yesterday FROM Transaction SINCE 2 days ago UNTIL 1 day ago
```

`SINCE`必須早於`UNTIL`。 否則，查詢會失敗，並顯示&#x200B;_SINCE必須早於UNTIL_。

+++

#### `WHERE` — 篩選列

+++檢視詳細資料

`WHERE`只會保留符合條件的列。 結合條件與`AND`、`OR`、`NOT`和括弧。 請參閱[運運算元和條件](#45-operators-and-conditions)。

```sql
SELECT count(*) FROM Transaction WHERE service = 'checkout' AND (status = 'Error' OR http.statusCode >= 500)
```

`WHERE`只能出現一次。 將所有條件都放進去。

+++

### 彙總函式

| 函數 | 引數 | 傳回 | 工作於 |
| -------------------------------------- | ----------------------------------- | ---------------------- | ------------------------------------- |
| [`apdex`](#fn-apdex) | 數值持續時間，臨界值`T` | 0-1 | 追蹤實體 |
| [`average` / `avg`](#fn-average) | 數值 | 與輸入相同的單位 | 任何 |
| [`count`](#fn-count) | `*`或屬性 | 列數 | 任何 |
| [`derivative`](#fn-derivative) | 數值 | 每秒變更 | 任何（需要`TIMESERIES`） |
| [`earliest`](#fn-latest) | 任何[，排名] | 屬性值 | 任何 |
| [`latest`](#fn-latest) | 任何[，排名] | 屬性值 | 任何 |
| [`max` / `min`](#fn-minmax) | 任何 | 與輸入相同的單位 | 任何 |
| [`percentage`](#fn-percentage) | `*`或屬性，`WHERE`條件 | % | 任何 |
| [`percentile`](#fn-percentile) | 數值，層級`p` | 與輸入相同的單位 | 任何 |
| [`percentOfTotal`](#fn-percentoftotal) | 數值 | % | 任何（無`TIMESERIES`） |
| [`rate`](#fn-rate) | 數值計數器 | 每秒 | 計數器實體（需要`TIMESERIES`） |
| [`stddev`](#fn-stddev) | 數值 | 與輸入相同的單位 | 任何 |
| [`sum`](#fn-sum) | 數值 | 與輸入相同的單位 | 任何 |
| [`uniqueCount`](#fn-uniquecount) | 任何 | 不同的值 | 任何 |

<a id="fn-apdex"></a>

#### `apdex(attribute, T)`

+++檢視詳細資料

使用屬性單位中的臨界值`T` （3}的毫秒），計算[Apdex](https://www.apdex.org/)從0 （受挫敗）到1 （滿意）的分數。`duration`

| 貯體 | 條件 | 粗細 |
| ---------- | ------------------------------------ | ------ |
| 滿意 | `duration ≤ T`且不是錯誤 | 1 |
| 容錯 | `T < duration ≤ 4T`且不是錯誤 | 0.5 |
| 受挫 | `duration > 4T`、**或任何錯誤** | 0 |

```sql
SELECT apdex(duration, 500) AS apdex FROM Transaction FACET service
```

- 僅適用於追蹤實體(`Transaction`、`AppRequest`、`Span`)。
- `T`必須大於0。 不允許`FILTER`。
- `Transaction`上的`apdex(duration, 500)`已透過長時段進行向上彙整加速。

+++

<a id="fn-average"></a>

#### `average(attribute)` / `avg(attribute)`

+++檢視詳細資料

數值屬性的算術平均值。

```sql
SELECT average(duration) AS avgMs FROM Transaction FACET service TIMESERIES AUTO
```

+++

<a id="fn-count"></a>

#### `count(*)` / `count(attribute)`

+++檢視詳細資料

- `count(*)`計入列。
- `count(attribute)`計算屬性為&#x200B;**不是NULL**&#x200B;的資料列。 對於字串屬性，遺漏的值為空白(`''`)而非NULL，因此仍會計算它們（請參閱[型別處理](#46-type-handling-blank-vs-null)）。

```sql
SELECT count(*) AS requests, count(http.statusCode) AS withStatusCode FROM Transaction
```

+++

<a id="fn-derivative"></a>

#### `derivative(attribute)`

+++檢視詳細資料

連續貯體之間`average(attribute)`的每秒變更： （此貯體−上一個貯體）÷貯體秒數。 結果可能是負數。 將其用於量測計，以檢視值是否正在增加或縮小。

```sql
SELECT derivative(value) AS heapGrowthPerSec FROM JvmHeapUsed WHERE host = 'publish-1' TIMESERIES 5 minutes
```

- 需要`TIMESERIES`，且必須是&#x200B;**僅限**&#x200B;彙總。 不允許`FILTER`。
- 第一個貯體沒有前置任務。

+++

<a id="fn-latest"></a>

#### `latest(attribute [, rankAttribute])` / `earliest(…)`

+++檢視詳細資料

- `latest(attr)`會傳回最近一列的值。 `earliest(attr)`會從最舊的資料列傳回值。
- 使用第二個引數時，該屬性會選擇資料列，而非時間。 `latest(traceId, duration)`傳回&#x200B;**最慢**&#x200B;要求的追蹤識別碼。 `earliest(traceId, duration)`傳回最快。

```sql
SELECT latest(value) AS current FROM MemoryUtilization FACET host
```

```sql
SELECT max(duration) AS slowestMs, latest(traceId, duration) AS slowestTrace FROM Transaction FACET name ORDER BY slowestMs DESC LIMIT 10
```

+++

<a id="fn-minmax"></a>

#### `max(attribute)` / `min(attribute)`

+++檢視詳細資料

最大或最小值。 適用於數字、字串和時間戳記。

```sql
SELECT min(duration) AS fastestMs, max(duration) AS slowestMs, max(timestamp) AS lastSeen FROM Transaction FACET service
```

+++

<a id="fn-percentage"></a>

#### `percentage(*, WHERE condition)`

+++檢視詳細資料

符合`condition`的資料列共用從0到100。

```sql
SELECT percentage(*, WHERE status = 'Error') AS errorPct FROM Transaction FACET service TIMESERIES AUTO
```

```sql
SELECT percentage(*, WHERE duration > 2000) AS slowPct FROM Transaction
```

`percentage`有自己的條件，因此不需要`FILTER`。

+++

<a id="fn-percentile"></a>

#### `percentile(attribute, p)`

+++檢視詳細資料

近似的第p個百分位數，其中`0 < p ≤ 100`。 每個層級使用一個`percentile`。

```sql
SELECT percentile(duration, 50) AS p50, percentile(duration, 95) AS p95, percentile(duration, 99) AS p99 FROM Transaction TIMESERIES AUTO
```

`Transaction`上的p50、p95和p99透過長視窗進行彙總加速。

+++

<a id="fn-percentoftotal"></a>

#### `percentOfTotal(attribute)`

+++檢視詳細資料

每個Facet資料列在數值屬性的&#x200B;**總和**&#x200B;中所佔的份額。 列數加起來為100%。

```sql
SELECT sum(duration) AS totalMs, percentOfTotal(duration) AS shareOfTime FROM Transaction FACET name LIMIT 15
```

不允許`TIMESERIES`。

+++

<a id="fn-rate"></a>

#### `rate(attribute)`

+++檢視詳細資料

**累積計數器**&#x200B;的每秒速率。 在每個貯體中，`rate`會取得每個數列的最高計數器讀數，並將其與上一個貯體做比較。 計數器重設設設為0，第一個貯體為0。

```sql
SELECT rate(value) AS bytesPerSec FROM NetworkReceiveRate FACET host TIMESERIES AUTO
```

- 僅適用於計數器實體（在[資料字典](#53-curated-metric-entities)中標示為✔）。
- 需要`TIMESERIES`，且必須是&#x200B;**僅限**&#x200B;彙總。 不允許`FILTER`。

>[!WARNING]
>
> 一律為`FACET host`，或篩選為單一主機。 許多主機間未分面的`rate()`會比較不同主機的計數器，並產生無意義的數字。 磁碟計數器是依裝置而定，因此`FACET host, device`是最精確的。

+++

<a id="fn-stddev"></a>

#### `stddev(attribute)`

+++檢視詳細資料

範例標準差。 將它與`average`搭配使用，以判斷延遲的一致程度。

```sql
SELECT average(duration) AS avgMs, stddev(duration) AS stddevMs FROM Transaction FACET name LIMIT 20
```

+++

<a id="fn-sum"></a>

#### `sum(attribute)`

+++檢視詳細資料

數值屬性的總數。

```sql
SELECT sum(callCount) AS calls FROM SpanBreakdown FACET segment LIMIT 20
```

+++

<a id="fn-uniquecount"></a>

#### `uniqueCount(attribute)`

+++檢視詳細資料

**確切**&#x200B;數目的不重複值。

```sql
SELECT uniqueCount(host) AS hosts, uniqueCount(traceId) AS traces FROM Transaction FACET service
```

+++

函式限制一覽：

| 限制 | 函數 |
| ------------------------------------- | ----------------------------------------------------------------------------------------- |
| 需要數值屬性 | `average`, `sum`, `percentile`, `stddev`, `percentOfTotal`, `rate`, `derivative`, `apdex` |
| 無`FILTER (WHERE …)` | `apdex`, `rate`, `derivative`, `percentage` |
| 需要`TIMESERIES`，僅彙總 | `rate`, `derivative` |
| 不與`TIMESERIES` | `percentOfTotal` |
| 僅限追蹤實體 | `apdex` |
| 僅限計數器實體 | `rate` |

### 運運算元和條件

| 運算子 | 含義 | 範例 |
| ------------------------- | -------------------------------------------------------------------------- | ---------------------------------------- |
| `=` | 等號 | `status = 'Error'` |
| `!=`, `<>` | 不等於 | `kind != 'Internal'` |
| `<`, `<=`, `>`, `>=` | 比較 | `http.statusCode >= 500` |
| `LIKE` | 萬用字元符合： `%` =任何字元，`_` =一個字元。 區分大小寫。 | `urlPath LIKE '/content/%'` |
| `NOT LIKE` | 否定的萬用字元 | `urlPath NOT LIKE '%.css'` |
| `IN (…)` | 符合1-100個值中的任一個 | `service IN ('checkout', 'cart')` |
| `NOT IN (…)` | 不符合任何值 | `http.method NOT IN ('OPTIONS', 'HEAD')` |
| `IS NULL` / `IS NOT NULL` | Null檢查 | `http.statusCode IS NOT NULL` |
| `AND`, `OR`, `NOT` | 邏輯 | `NOT (status = 'Ok')` |
| `( … )` | 分組 | `(a = 1 OR b = 2) AND c = 3` |

```sql
SELECT count(*) FROM Transaction WHERE (status = 'Error' OR duration > 2000) AND NOT host LIKE '%dispatcher%' AND http.method IN ('GET', 'POST')
```

> **警告**
>
> `attr = NULL`為錯誤。 使用`attr IS NULL`。

### 型別處理：空白與NULL

目錄中的每個屬性都有型別： `string`、`number`、`boolean`或`timestamp`。 SQL型別檢查函式（例如，`average(service)`被拒絕）。 不需要或沒有可用的型別轉換函式。

> **重要**
>
> **空白與NULL。** 大部分字串屬性來自OpenTelemetry屬性對應，其中&#x200B;_遺失_&#x200B;索引鍵會讀取為空字串`''`，而不是`NULL`。 若要只保留具有屬性的範圍，請使用&#x200B;**`attr != ''`**&#x200B;而非`IS NOT NULL`。 數值屬性（例如`http.statusCode`）不存在時實際上是`NULL`，因此`IS NOT NULL`適用於它們。

```sql
SELECT count(*) AS jcrCalls FROM Span WHERE jcr.operation != '' FACET jcr.operation
```

### 將SQL轉譯為SQL

| SQL | SQL |
| ------------------------------------------------------------ | ------------------------------------------------------------------- |
| `WHERE ts > now() - INTERVAL 1 HOUR` | 沒有內容；時間選擇器（或`SINCE 1 hour ago`）設定視窗 |
| `GROUP BY service` | `FACET service` |
| `GROUP BY toStartOfInterval(ts, INTERVAL 5 MINUTE), service` | `FACET service TIMESERIES 5 minutes` |
| `quantile(0.95)(duration)` | `percentile(duration, 95)` |
| `countIf(status = 'Error')` | `count(*) FILTER (WHERE status = 'Error')` |
| `100 * countIf(x) / count()` | `percentage(*, WHERE x)` |
| `count(DISTINCT host)` | `uniqueCount(host)` |
| `argMax(traceId, duration)` | `latest(traceId, duration)` |
| `x / sum(x) OVER ()` | `percentOfTotal(x)` |
| `ORDER BY p95 DESC LIMIT 20` | `ORDER BY p95 DESC LIMIT 20` |
| `HAVING count() > 100` | _不支援_。 改為排序和`LIMIT`。 |
| `JOIN` /子查詢 | _不支援_。 使用&#x200B;**公式** Widget。 |
| `CASE WHEN` | _不支援_。 使用`FILTER`欄，或`WHERE … LIKE` + `FACET`。 |

### 將現有儀表板匯入到Insights

如果您知道現有查詢，以下是移轉詳細資料

| 現有查詢 | SQL | 備註 |
| ------------------------------------------- | ------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------- |
| `FROM Transaction SELECT count(*)` | 相同 | 支援`FROM`第一順序 |
| `SINCE 1 day ago` | 相同 | 忽略儀表板；選擇器獲勝 |
| `TIMESERIES AUTO` / `MAX` / `5 minutes` | 相同 | `AUTO`樓層是2分鐘 |
| `FACET appName` | `FACET appName` / `FACET service` |                                                                                  |
| `percentage(count(*), WHERE error IS true)` | `percentage(*, WHERE status = 'Error')` |                                                                                  |
| `filter(count(*), WHERE …)` | `count(*) FILTER (WHERE …)` | SQL標準表單 |
| `apdex(duration, t: 0.5)` | `apdex(duration, 500)` | `duration`在&#x200B;**毫秒**&#x200B;內 |
| `latest(x)` / `earliest(x)` | 相同 | 加上排名表單： `latest(x, rankAttr)` |
| `uniqueCount(x)` | 相同 | 精確 |
| `rate(count(*), 1 minute)` | `count(*)`與`TIMESERIES 1 minute` | SQL `rate()`僅適用於計數器 |
| `derivative(x, 1 minute)` | `derivative(x)` + `TIMESERIES` | 每秒 |
| `uniques(x)` | `count(*) … FACET x LIMIT n` |                                                                                  |
| `keyset()` / `SHOW EVENT TYPES` | [探索查詢](../examples/query-cookbook.md#data-discovery)或查詢指南面板 |                                                                                  |
| `histogram(duration, …)` | `FILTER`個值區 | 請參閱[教學課程3](../tutorials/tutorial-3.md#build-a-latency-histogram) |
| `FACET CASES (…)` | `FILTER`欄 | 請參閱[教學課程3](../tutorials/tutorial-3.md#group-values-into-cases-with-filter) |
| `COMPARE WITH 1 week ago` | 不支援 | 變更時間選擇器，或透過API使用`SINCE … UNTIL …` |
| `RLIKE` | `LIKE` / `IN` |                                                                                  |
| `SELECT`中的數學/巢狀彙總 | **公式** Widget |                                                                                  |
| `WITH TIMEZONE`, `EXTRAPOLATE`, `SLIDE BY` | 不支援 |                                                                                  |
