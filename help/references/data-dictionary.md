---
source-git-commit: 4949fcad799260aa73354a8c0b9d79e6394cb257
workflow-type: tm+mt
source-wordcount: '585'
ht-degree: 20%
---
## 資料字典 {#data-dictionary}

這是您可以查詢的每個實體和屬性。 即時目錄（含說明）也在「瀏覽」和「新增Widget」對話方塊的&#x200B;**查詢指南**&#x200B;面板中。

### 追蹤實體

| 實體 | 包含 | 固定範圍 | 統計 |
| ------------- | --------------------------------------------------------- | ---------------------------- | ------------------------------------------ |
| `Transaction` | 傳入伺服器要求 | `kind = Server` | 1米/ 5米/ 1小時（[效能](#performance)） |
| `AppRequest` | 傳入伺服器&#x200B;**和**&#x200B;訊息消費者要求 | `kind IN (Server, Consumer)` | — |
| `Span` | 每個範圍（伺服器、使用者端、生產者、消費者、內部） | — | — |

<details open>
<summary><strong>追蹤屬性</strong> （在所有三個實體上相同）</summary>

| 屬性 | 類型 | 說明 |
| --------------------------------------------------- | ----------- | ----------------------------------------------------------------------------------------------- |
| `timestamp` | timestamp | 跨越開始時間 |
| `traceId`, `spanId`, `parentSpanId` | 字串 | 識別碼（根範圍有`parentSpanId = ''`） |
| `name` | 字串 | 作業或異動名稱 |
| `kind` | 字串 | `Server`, `Client`, `Producer`, `Consumer`, `Internal` |
| `service` | 字串 | 正在發出服務名稱 |
| `serviceKey` | 字串 | 有效的服務識別（當`service`為空白或粗略父級時，將退回`appName`） |
| `appName` | 字串 | Observability Insights應用程式名稱 |
| `appInstance` | 字串 | 每個執行個體身分：依作者/發佈主機首碼、應用程式名稱、服務 |
| `hostTierPrefix` | 字串 | 主機名稱在其`author` / `publish` / `dispatcher`角色尾碼處被截斷 |
| `duration` | 數字（毫秒） | 跨距持續時間（毫秒） |
| `status` | 字串 | `Ok`, `Error`, `Unset` |
| `statusMessage` | 字串 | 狀態說明 |
| `host` | 字串 | 主機名稱 |
| `environment` | 字串 | 部署環境 |
| `region` | 字串 | 雲端區域 |
| `cloud.provider` | 字串 | AWS / AZURE / GCP |
| `http.method` | 字串 | `GET`、`POST`、... |
| `http.statusCode` | 數字 | HTTP回應狀態 |
| `http.route` | 字串 | 標準化路由範本 |
| `url` | 字串 | 完整URL |
| `urlPath` | 字串 | URL (`/content/site/en.html`)的路徑部分 |
| `externalDomain` | 字串 | 輸出URL的網域（用於使用者端範圍） |
| `peer.service` | 字串 | 已呼叫下游服務 |
| `db.system`, `db.operation`, `db.namespace` | 字串 | 資料庫技術、作業和結構描述 |
| `messaging.system`, `messaging.destination` | 字串 | Kafka、RabbitMQ...以及主題或佇列 |
| `rpc.system`, `rpc.method` | 字串 | gRPC和類似專案 |
| `error.type` | 字串 | 例外狀況或錯誤型別 |
| `sling.script`, `sling.servlet`, `sling.requestUri` | 字串 | AEM Sling詳細資料 |
| `jsp.path` | 字串 | AEM JSP已執行 |
| `jcr.operation`, `jcr.path` | 字串 | AEM JCR存放庫作業和路徑 |
| `graphql.operation` | 字串 | GraphQL操作名稱 |

</details>

### 一般量度實體

| 實體 | 包含 |
| ------------- | -------------------------------------------------- |
| `MetricGauge` | 每個時間點OpenTelemetry量測軌範例 |
| `MetricSum` | 每個累積或差異OpenTelemetry總和範例 |

將它們用於&#x200B;**探索**，或用於沒有已組織實體的量度： `WHERE metric = 'jvm.gc.duration'`。

<details open>
<summary><strong>量度屬性</strong> （由所有量度實體共用）</summary>

| 屬性 | 類型 | 說明 |
| ----------------------------------------------------------------------- | --------- | ------------------------------------------------------- |
| `timestamp` | timestamp | 範例時間 |
| `metric` | 字串 | OpenTelemetry測量結果名稱 |
| `value` | 數字 | 範例值（在已組織的實體上轉換；請參閱下文） |
| `unit` | 字串 | 量度單位 |
| `service`, `appName`, `host`, `environment`, `region`, `cloud.provider` | 字串 | 資源身分 |
| `state` | 字串 | CPU模式或記憶體狀態 |
| `cpu`, `device`, `direction`, `mountpoint` | 字串 | 每個CPU、每個裝置、I/O方向、檔案系統 |
| `pool` | 字串 | JVM記憶體集區或連線集區 |
| `topic` | 字串 | Kafka主題 |
| `process.pid`, `process.name`, `process.owner` | 字串 | 處理程式識別 |

</details>

### 已組織的量度實體

每個已組織的實體會以正確的篩選器和單位轉換來包裝一個OpenTelemetry量度，因此`value`已經很有意義。 **計數器**&#x200B;個實體(✔)儲存累積總計。 使用`rate(value)`和`TIMESERIES`查詢它們。

| 實體 | `value`表示 | 單位 | 計數器 | 備註 |
| ------------------------------------------------------------------------------ | ----------------------------------- | --------- | ------- | -------------------------------------------- |
| `CpuUtilization` | CPU忙碌 | % |         | 依主機為`average(value)`進行1分鐘統計 |
| `CpuUserPercent` / `CpuSystemPercent` / `CpuIowaitPercent` / `CpuStealPercent` | CPU時間（依模式） | % |         |                                              |
| `MemoryUtilization` | 使用的記憶體 | % |         | `average(value)`的1分鐘統計 |
| `LoadAverage1m` / `LoadAverage5m` / `LoadAverage15m` | 平均載入 | — |         |                                              |
| `LoadAverageWithFallback` | 5百萬載入，下降至1百萬載入 | — |         |                                              |
| `DiskUtilization` | 每次掛載使用的磁碟 | % |         | Facet by `mountpoint` |
| `DiskFreeBytes` | 可用根檔案系統 | 位元組 |         |                                              |
| `DiskReadRate` / `DiskWriteRate` | 磁碟輸送量 | 位元組/秒 | ✔ |                                              |
| `DiskReadOpsRate` / `DiskWriteOpsRate` | 磁碟IOPS | ops/s | ✔ |                                              |
| `NetworkReceiveRate` / `NetworkTransmitRate` | eth0輸送量 | 位元組/秒 | ✔ |                                              |
| `NetworkReceiveRateAllDevices` / `NetworkTransmitRateAllDevices` | 所有裝置 | 位元組/秒 | ✔ |                                              |
| `NetworkIoRate` | 所有方向與裝置組合 | 位元組/秒 | ✔ |                                              |
| `NetworkReceiveErrorRate` / `NetworkTransmitErrorRate` | 網路錯誤 | 錯誤/秒 | ✔ |                                              |
| `NetworkReceivePacketRate` / `NetworkTransmitPacketRate` | 封包 | 封包/秒 | ✔ |                                              |
| `ProcessCpuUtilization` | 每個程式的CPU | % |         | Facet by `process.name` |
| `ProcessMemoryUsage` | 每個處理序的記憶體 | MB |         |                                              |
| `JvmHeapUsed` / `JvmHeapLimit` | JVM棧積 | MB |         | 每個記憶體集區報告 |
| `JvmThreadCount` | JVM執行緒 | — |         |                                              |
| `JvmCpuRecentUtilization` | JVM程式CPU | % |         | 1分鐘統計 |
| `DispatcherRequestRate` | AEM Dispatcher HTTP請求 | 需要 | ✔ |                                              |

&#x200B;> [!NOTE]
> 每個記憶體集區&#x200B;**報告JVM記憶體量度**，因此`JvmHeapUsed` / `JvmHeapLimit`上的`average(value)`是跨集區的平均值。 請使用`FACET host, pool`來取得每個集區的正確檢視。

### `SpanBreakdown`：預先彙總的交易構成

`SpanBreakdown`保留每分鐘每個交易的子範圍組合。 這是在內嵌時計算，因此查詢起來很便宜。

| 屬性 | 類型 | 說明 |
| --------------- | --------- | ---------------------------------------------------------------------- |
| `timestamp` | timestamp | 一分鐘儲存貯體 |
| `service` | 字串 | 服務 |
| `rootOperation` | 字串 | 父級交易名稱 |
| `segment` | 字串 | 子範圍名稱 |
| `callCount` | 數字 | 貯體中此區段的呼叫 |
| `avgDurationMs` | 數字 | 貯體中的平均區段持續時間（手動將單位設定為`ms`） |
