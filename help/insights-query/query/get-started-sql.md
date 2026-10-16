---
title: 開始使用SQL
description: 瞭解如何使用SQL查詢Observability Insights遙測。
source-git-commit: 09c8edf64e744ff0c16acd0d5905fc6796490f82
workflow-type: tm+mt
source-wordcount: '684'
ht-degree: 1%
---

# 快速入門

**Adobe Observability Insights Query Language (SQL)** — 在Adobe Observability Insights中查詢、圖表和儀表板。

|                |                                                                  |
| -------------- | ---------------------------------------------------------------- |
| **套用至** | 深入分析UI → **探索**&#x200B;和&#x200B;**儀表板** （自訂儀表板） |
| **客群** | 瞭解SQL的工程師、SRE和進階使用者 |

>[!TIP]
>
> 每個標示為`sql`的程式碼區塊都是SQL查詢，您可以將其直接貼到&#x200B;**探索→的SQL**&#x200B;或&#x200B;**儀表板→新增Widget→SQL**。

## 開始使用SQL：遙測的語言 {#get-started-with-sql-the-language-of-your-telemetry}

前瞻分析查詢語言是類似於SQL的查詢語言，用於您的Observability Insights資料。 這是每個自訂儀表板Widget和&#x200B;**瀏覽**&#x200B;頁面背後的相同語言，因此您可以建立團隊需要的檢視。

SQL與&#x200B;**個實體**&#x200B;搭配使用，這些實體是好記的邏輯檢視，例如`Transaction`、`Span`或`CpuUtilization`。 您絕對不會直接查詢儲存空間表格。 「可觀察性深入分析」會將每個SQL查詢編譯為安全、引數化的倉儲查詢。 它一律限定查詢租使用者的範圍和所選的時間範圍。

```sql
SELECT count(*) AS requests FROM Transaction TIMESERIES AUTO
```

### 您可以使用SQL做什麼

使用SQL可以：

- **建置圖表**&#x200B;的輸送量、錯誤率、延遲、Apdex、CPU、記憶體、JVM棧積、磁碟和網路隨時間變化。
- **回答特定問題**，例如「發佈層級上的哪些URL路徑的p95值最差？」 或「哪個下游網域失敗？」
- **從KPI向下鑽研至個別要求與追蹤ID，以調查事件**。
- **使用KPI、趨勢、表格、公式、附註、區段標題和變數建立儀表板**。
- **使用Sling、JSP、JCR、Dispatcher和AEM屬性來分析GraphQL工作負載** (Adobe Experience Manager)。

SQL同時支援&#x200B;**事件查詢** （傳回個別要求列）和&#x200B;**彙總查詢** （傳回總計、百分位數、比率、劃分和時間序列）。

>[!TIP]
>
> **儀表板→建立儀表板**&#x200B;中的範本完全由SQL建置。 開啟任何具有鉛筆圖示的範本Widget，以檢視其查詢並從中學習。

### 您可以查詢的內容

SQL可以查詢四個資料系列。 完整清單位於[資料字典](/help/references/data-dictionary.md)中。

| 家庭 | 實體 | 常見問題 |
| ------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------ | -------------------------------------------------------------------- |
| **追蹤(APM)** | `Transaction` （傳入要求）、`AppRequest` （要求+訊息消費者）、`Span` （每個範圍） | 輸送量、延遲、錯誤、Apdex、下游呼叫、AEM程式碼路徑 |
| **已組織的主機和執行階段量度** | `CpuUtilization`、`MemoryUtilization`、`LoadAverage5m`、`DiskUtilization`、`NetworkReceiveRate`、`JvmHeapUsed`、`JvmThreadCount`、`DispatcherRequestRate`等 | 飽和度、容量、JVM健全狀態、Dispatcher負載 |
| **一般量度** | `MetricGauge`, `MetricSum` | 依名稱、測量結果探索分類的任何OpenTelemetry測量結果 |
| **預先彙總的跨區組合** | `SpanBreakdown` | 構成交易的子作業 |

>[!NOTE]
>
> 記錄、警示定義和詳細目錄關係無法使用。

### SQL的使用位置

| 位置 | 如何開啟 | 用途 |
| ----------------------- | ------------------------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| **探索** | 左側導覽→**瀏覽** | 臨機分析。 使用&#x200B;**視覺化產生器**&#x200B;或&#x200B;**SQL**&#x200B;索引標籤，**驗證**，然後&#x200B;**執行** (⌘/Ctrl + Enter)。 包含&#x200B;**歷程記錄**&#x200B;和&#x200B;**已儲存**&#x200B;的查詢以及&#x200B;**查詢指南**&#x200B;面板。 |
| **儀表板** | 左導覽→**儀表板**→開啟或建立儀表板→**新增Widget** | 結合到儀表板中的持續性圖表、KPI和表格。 |
| **公式Widget** | **將Widget新增→公式** | 結合2到6個SQL查詢與算術（比率、淨餘空間、燒錄率）。 |
| **儀表板變數** | 從查詢&#x200B;**→取的**+變數 | 從SQL `FACET`查詢的不同值填入選取器。 |
| **查詢檢查** | 將游標暫留在圖表→檢查查詢 | 檢視內建或儀表板圖表背後的SQL。 |

### SQL查詢結構

SQL遵循以子句為基礎的語法。 此查詢會計算所選視窗中的所有入站請求：

```sql
SELECT count(*) FROM Transaction
```

以下是所有可用的條款：

```text
SELECT  function(attribute) [FILTER (WHERE condition)] [AS alias] [, ...]
        | attribute [AS alias] [, ...]
FROM    Entity
  [WHERE      condition [AND | OR ...]]
  [FACET      attribute [, attribute ...]]        -- up to 5
  [SINCE      n unit [AGO]]
  [UNTIL      NOW | n unit [AGO]]
  [TIMESERIES AUTO | MAX | n unit]
  [ORDER BY   alias [ASC | DESC]]
  [LIMIT      number]                              -- 1 to 5000
```

SQL遵循的規則如下：

| 規則 | 詳細資料 |
| ---------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **必要的子句** | 需要`SELECT`和`FROM`。 其他每個子句都是選擇性的。 您可以以下列任一專案開頭： `FROM Transaction SELECT count(*)`也是有效的。 |
| **子句順序** | `FROM`之後的子句可以任何順序出現。 每個子句只能出現一次。 |
| **查詢大小** | 查詢大小必須等於或少於4 KB。 |
| **區分大小寫** | 關鍵字、函式名稱、實體名稱和屬性名稱是&#x200B;**不區分**&#x200B;大小寫。 字串值&#x200B;**有**&#x200B;區分大小寫(`'Error'` ≠ `'error'`)。 |
| **字串** | 使用單引號： `WHERE status = 'Error'`。 若要包含引號，請加倍引號： `'O''Brien'`。 不接受雙引號。 |
| **常值** | 字串、數字（包括負數與小數）、`TRUE`、`FALSE`、`NULL`。 |
| **屬性名稱** | 名稱可以包含字母、數字、`_`和`.` （例如`http.statusCode`）。 不需要反引號。 |
| **彙總與屬性** | 查詢會選取原始屬性（事件清單）或彙總，但絕不會同時選取兩者。 若要分組，請使用`FACET`。 |
| **時間範圍** | 預設為過去一小時。 最長31天。 在控制面板和Explore中，**時間選擇器**&#x200B;會設定視窗（請參閱[時間視窗](how-to-query-sql.md#time-window-precedence)）。 |
| **不支援** | 分號、註解、子查詢、聯結、`SELECT *`、`SELECT`、`HAVING`、`COMPARE WITH`中的屬性之間的數學。 請參閱[SQL沒有做什麼](sql-query-examples.md#what-sql-doesnt-do)以取得替代方案。 |
