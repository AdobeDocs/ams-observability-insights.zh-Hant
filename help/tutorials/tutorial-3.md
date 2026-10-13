---
source-git-commit: 263ba9e9b3eb1c6e4dd253ca03bd0ce8954c588b
workflow-type: tm+mt
source-wordcount: '888'
ht-degree: 3%
---
## 教學課程3：推進您的儀表板 {#tutorial-3-advance-your-dashboarding}

在本教學課程中，您將建立生產等級儀表板的元件：使用者體驗分數、計數器比率、案例分組、長條圖、具有臨界值的KPI圖磚、區段、公式Widget和變數。

**在本教學課程中：** [8.1 apdex](#81-score-user-experience-with-apdex) · [8.2 rate](#82-turn-counters-into-rates-with-rate) · [8.3案例](#83-group-values-into-cases-with-filter) · [8.4長條圖](#84-build-a-latency-histogram) · [8.5 KPI圖磚和區段](#85-build-kpi-tiles-and-sections) · [8.6 Formula](#86-combine-queries-with-formula-widgets) · [8.7變數](#87-make-dashboards-reusable-with-variables) · [8.8放在一起](#88-put-it-together)

### 使用`apdex`為使用者體驗評分

Apdex會從0到1將延遲和錯誤變成1分。 在500毫秒臨界值下，要求在500毫秒或以下&#x200B;_滿足_，_容忍_&#x200B;最多2,000毫秒，以及&#x200B;_受挫感_&#x200B;高於該&#x200B;**或如果失敗**：

```sql
SELECT apdex(duration, 500) AS apdex FROM Transaction
```

依服務分析趨勢：

```sql
SELECT apdex(duration, 500) AS apdex FROM Transaction FACET service TIMESERIES AUTO LIMIT 8
```

將它放入臨界值為`0 → Red`、`0.7 → Amber`、`0.85 → Green`的&#x200B;**數字**&#x200B;介面工具中。 最高的相符波段會獲勝，因此低分會維持紅色。

### 8.2使用`rate`將計數器轉換為速率

計數器實體（[資料字典](#53-curated-metric-entities)中的✔）會儲存不斷增加的總計。 `rate(value)`將其轉換為每秒的值。 它需要`TIMESERIES`，而且必須是唯一的彙總：

```sql
SELECT rate(value) AS rxBytesPerSec FROM NetworkReceiveRate FACET host TIMESERIES AUTO
```

```sql
SELECT rate(value) AS writeBytesPerSec FROM DiskWriteRate FACET host, device TIMESERIES AUTO LIMIT 10
```

&#x200B;> [!WARNING]
> 一律以`host` （和磁碟的`device`）面向`rate()`，或篩選為單一主機。 若沒有這些，來自不同主機的讀數會混合在一起，而結果毫無意義。

若要從追蹤取得&#x200B;**請求**&#x200B;速率，請改為以固定值區計算請求。 這會提供每分鐘的請求數：

```sql
SELECT count(*) AS requestsPerMinute FROM Transaction TIMESERIES 1 minute
```

### 將值分組為具有`FILTER`的大小寫

NRQL的`FACET CASES`會將資料列分組到具名儲存貯體中。 在SQL中，每個案例寫入一個`FILTER`資料行。 以下是每個服務的狀態類別：

```sql
SELECT count(*) FILTER (WHERE http.statusCode < 400) AS ok, count(*) FILTER (WHERE http.statusCode >= 400 AND http.statusCode < 500) AS clientErrors, count(*) FILTER (WHERE http.statusCode >= 500) AS serverErrors FROM Transaction FACET service LIMIT 20
```

隨著時間推移，相同的構想會逐漸生效，每個案例各一行：

```sql
SELECT count(*) FILTER (WHERE urlPath LIKE '/content/%') AS pages, count(*) FILTER (WHERE urlPath LIKE '/bin/%') AS servlets, count(*) FILTER (WHERE urlPath LIKE '/graphql/%') AS graphql FROM Transaction TIMESERIES AUTO
```

### 建立延遲長條圖

SQL沒有`histogram()`函式。 使用`FILTER`自行建立值區，並將結果轉譯為&#x200B;**列**&#x200B;或&#x200B;**表格** Widget：

```sql
SELECT count(*) FILTER (WHERE duration <= 100) AS 'le 100ms', count(*) FILTER (WHERE duration > 100 AND duration <= 250) AS '100-250ms', count(*) FILTER (WHERE duration > 250 AND duration <= 500) AS '250-500ms', count(*) FILTER (WHERE duration > 500 AND duration <= 1000) AS '0.5-1s', count(*) FILTER (WHERE duration > 1000 AND duration <= 2000) AS '1-2s', count(*) FILTER (WHERE duration > 2000) AS 'over 2s' FROM Transaction
```

新增`FACET service`以並排比較數個服務的分佈。

### 建立KPI圖磚和區段

良好的儀表板會從一列KPI圖磚開始，然後將詳細圖表分組為區段。

1. 新增四個&#x200B;**Number** Widget，每3個欄寬：

   | 標題 | 查詢 | 單位 | 臨界值 |
   | -------- | --------------------------------------------------------------------------- | ---- | ---------------------------------------- |
   | 請求 | `SELECT count(*) AS requests FROM Transaction` |      |                                          |
   | 錯誤% | `SELECT percentage(*, WHERE status = 'Error') AS errorPct FROM Transaction` | `%` | `0 → Green`, `1 → Amber`, `5 → Red` |
   | p95 | `SELECT percentile(duration, 95) AS p95 FROM Transaction` | `ms` | `0 → Green`, `800 → Amber`, `2000 → Red` |
   | Apdex | `SELECT apdex(duration, 500) AS apdex FROM Transaction` |      | `0 → Red`, `0.7 → Amber`, `0.85 → Green` |

2. 新增&#x200B;**文字** Widget，勾選&#x200B;_將此設為區段標題_，並呼叫它&#x200B;**延遲**。 在下方新增延遲圖。
3. 新增另一個區段標頭&#x200B;**ERRORS**，後面接著錯誤圖表。 收合帶有V形箭號的區段以集中在另一個區段上。
4. 針對不同的對象使用&#x200B;**標籤**，例如&#x200B;_總覽_、_AEM_、_基礎架構_。 只有可見的索引標籤會執行其查詢。

### 結合查詢與公式Widget

SQL無法在`SELECT`中進行數學運算，而且沒有子查詢。 **公式Widget**&#x200B;填補該空白：它們執行2-6個具名查詢，並將其結果與`+ - * / ( )`結合。

**範例：每個主機在一段時間內的要求** （公式，行）

1. **→公式**&#x200B;新增Widget。
2. 為第一個查詢`requests`及第二個`hosts`命名。
3. 輸入查詢和公式，然後按一下&#x200B;**執行公式**。

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

**範例：在資料庫** （公式、數字、單位`%`）中花費的時間共用

查詢`db`：

```sql
SELECT sum(duration) AS v FROM Span WHERE db.system != ''
```

查詢`server`：

```sql
SELECT sum(duration) AS v FROM Transaction
```

公式：

```text
(db / server) * 100
```

公式查詢的規則：

| 規則 | 詳細資料 |
| --------------- | -------------------------------------------------------------------------------------------------------------------- |
| 名稱 | 字母、數字和`_`，以字母或`_`開頭；最多40個字元；唯一 |
| 運算式 | `+ - * / ( )`、數字和查詢名稱；最多500個字元 |
| 相同形狀 | 所有查詢都是純總計（數字）、**或**&#x200B;都是`TIMESERIES`，具有&#x200B;**相同的儲存貯體大小** （行/區域/列） |
| 每個量度 | 每個查詢只傳回一個彙總，不傳回`FACET` |
| 缺少資料 | 缺少值或除以零會產生空點，永遠不會`Infinity`或`NaN` |

&#x200B;> [!TIP]
> 在時間序列公式的每一個查詢中使用`TIMESERIES AUTO`。 由於查詢共用一個視窗，因此它們一律會解析為相同的貯體大小。

### 讓儀表板可重複使用變數

變數可讓一個儀表板提供許多服務、層級或臨界值。 每個變數都是變數列中的選取器。 在Widget標題和文字Widget中將其參照為`{{name}}` （在下列限制修正後，在Widget SQL中參照）。

**建立變數** （_組織和超級管理員_）：按一下&#x200B;**+變數**。

| 欄位 | 備註 |
| ----------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **名稱** | 字母、數字和`_`，以字母或`_`開頭。 必須是唯一的。 已用作`{{name}}`。 |
| **標籤** | 選擇器顯示的內容（選擇性）。 |
| **類型** | **自訂清單** （每行一個值）、**來自查詢** （具有一個`FACET`的SQL查詢；其不同的值變成選項）或&#x200B;**文字方塊** （任意文字）。 |
| **允許選取多個值** | 展開至逗號分隔清單。 與`IN (…)`搭配使用。 |
| **值是數字** | 插入未引號的值（`500`而非`'500'`）。 |

**替代的運作方式**

| 選擇 | `{{svc}}`變成 |
| ---------------------------------- | ------------------------------------------------------- |
| 單一字串`checkout` | `'checkout'` |
| 多個值`checkout`， `cart` | `'checkout', 'cart'` |
| 數字`500` | `500` |
| 包含`'`的值 | 已逸出為`''` |
| 未選取任何專案 | Widget會等待並顯示&#x200B;_選取{{svc}}_&#x200B;的值 |

```text
WHERE service = {{svc}}            single select
WHERE service IN ({{svc}})         multiple select
WHERE duration > {{slowMs}}        numeric text box
```

對於查詢導向的變數，**一律新增`LIMIT`**。 多面向查詢預設只會傳回前10個值。 此清單列出最多200項服務：

```sql
SELECT count(*) FROM Transaction FACET service LIMIT 200
```

當租使用者或時間選擇器變更時，選項會重新整理，而且會依字母順序排序。 當目前選取範圍不再有效時，會自動選取第一個值（單選）或所有值（多選）。 變數的查詢無法參考其他變數。

&#x200B;> [!WARNING]
> **目前的組建限制。** 儀表板格線中會發生替代，但&#x200B;**新增Widget**&#x200B;預覽和&#x200B;**伺服器端儲存檢查**&#x200B;會收到原始`{{name}}`權杖並使用&#x200B;_非預期的字元&#39;&lbrace;&#39;_&#x200B;加以拒絕。 實際操作中：
>
> - **介面工具集標題**&#x200B;和&#x200B;**文字介面工具集**&#x200B;中的`{{variables}}`工作端對端。
> - 在&#x200B;**匯入**&#x200B;之後，**介面工具集SQL**&#x200B;內的`{{variables}}`會正確轉譯，但是當這些Widget存在時，儀表板無法&#x200B;**儲存**。
>
> 在修正此問題之前，請先建置具有常值篩選器的每個內容Widget （例如`WHERE service = 'checkout'`），並使用&#x200B;**複製**&#x200B;來複製它們。

### 整合在一起

您現在已在[儀表板Blueprint](#dashboard-blueprints)中使用每個建置區塊：具有臨界值、區段、百分位數、Apdex、比率、大小寫欄、長條圖、公式和變數的KPI圖磚。 藍圖說明如何將它們結合為完整的黃金訊號、錯誤分類、AEM、基礎結構和JVM儀表板。
