---
title: 教學課程1：您的第一個查詢
description: 針對可觀察性深入分析遙測編寫您的第一個SQL查詢。
source-git-commit: 143a9567da06f0137ef690795cfbc41bda1780ca
workflow-type: tm+mt
source-wordcount: '576'
ht-degree: 0%
---

## 教學課程1：您的第一個查詢 {#tutorial-1-your-first-queries}

歡迎使用SQL。 在本教學課程中，您將撰寫第一個查詢、控制時間範圍、繪製趨勢、篩選和分組結果，並將第一個圖表放在儀表板上。 每個步驟都建立在上一個步驟之上。

**在本教學課程中：** [在您開始之前](#before-you-begin) · [6.1查詢事件](#61-query-events-with-select-from-and-limit) · [6.2彙總](#62-aggregate-your-data) · [6.3數個彙總](#63-return-several-aggregates-at-once) · [6.4時間視窗](#64-set-the-time-window) · [6.5時間序列](#65-plot-trends-with-timeseries) · [6.6位置](#66-filter-with-where) · [6.7 FACET](#67-group-with-facet) · [6.8儲存至儀表板](#68-save-your-query-to-a-dashboard)

### 開始之前

- 您需要存取可觀察性深入分析，以及至少一個傳送追蹤或量度的客戶租使用者。
- 在租使用者切換器中選取該&#x200B;**客戶**。 選取&#x200B;**所有帳戶**&#x200B;時，SQL未執行。
- 開啟&#x200B;**瀏覽**&#x200B;並切換至&#x200B;**SQL**&#x200B;索引標籤。 在每次查詢後使用&#x200B;**執行** (⌘/Ctrl + Enter)。

### 具有`SELECT`、`FROM`和`LIMIT`的查詢事件

每個查詢都需要兩個子句： `SELECT`指出要傳回的專案，而`FROM`指出要讀取的實體。 從傳入要求開始：

```sql
SELECT timestamp, service, name, duration FROM Transaction
```

這是&#x200B;**事件查詢**。 每一列都是一個請求。 若沒有`LIMIT`，您最多會有100列。 要求更少：

```sql
SELECT timestamp, service, name, duration FROM Transaction LIMIT 10
```

列會依第一欄遞減排序，所以將`timestamp`放在第一位，您最新的請求就會顯示在最上方。

>[!NOTE]
>
> SQL沒有`SELECT *`。 列出屬性可讓結果保持小規模且易於讀取。 [資料字典](#data-dictionary)和編輯器中的自動完成會顯示可用的內容。

### 彙總您的資料

大部分的儀表板不會顯示個別事件。 內容摘要說明。 以&#x200B;**彙總函式**&#x200B;取代屬性：

```sql
SELECT count(*) FROM Transaction
```

這是視窗中的傳入要求數目。 請再試一次：

```sql
SELECT average(duration) FROM Transaction
```

```sql
SELECT max(duration) FROM Transaction
```

```sql
SELECT uniqueCount(host) FROM Transaction
```

### 一次傳回多個彙總

以逗號分隔彙總，並以`AS`命名每個彙總：

```sql
SELECT count(*) AS requests, average(duration) AS avgMs, max(duration) AS maxMs FROM Transaction
```

>[!TIP]
>
> 別名會變成欄標題和圖例標籤，而它們就是您在`ORDER BY`中參照的內容。 養成命名每個彙總的習慣。

### 設定時間視窗

依預設，查詢涵蓋&#x200B;**前一個小時**。 使用`SINCE`進一步回顧：

```sql
SELECT count(*) AS requests FROM Transaction SINCE 1 day ago
```

使用`UNTIL`結束過去的視窗。 這會計算昨天從48到24小時前的流量：

```sql
SELECT count(*) AS requests FROM Transaction SINCE 2 days ago UNTIL 1 day ago
```

單位可以是`second`、`minute`、`hour`、`day`或`week`。 視窗最多可為31天。

>[!IMPORTANT]
>
> 在&#x200B;**瀏覽**&#x200B;和&#x200B;**儀表板**&#x200B;中，頁面頂端的&#x200B;**時間選擇器**&#x200B;會控制視窗並覆寫`SINCE`/`UNTIL`。 若要檢視其他期間，請變更選擇器。 本指南的其餘部分則因此將`SINCE`保留在儀表板查詢之外。

### 使用`TIMESERIES`繪製趨勢

新增`TIMESERIES`以隨著時間將單一數字轉換為行：

```sql
SELECT count(*) AS requests FROM Transaction TIMESERIES AUTO
```

`AUTO`選擇儲存貯體大小，因此圖表大約有120點（1小時視窗為2分鐘，24小時視窗為15分鐘）。 您也可以明確設定貯體：

```sql
SELECT average(duration) AS avgMs FROM Transaction TIMESERIES 5 minutes
```

### 使用`WHERE`篩選

`WHERE`僅保留您關心的資料列。 僅計算失敗的請求：

```sql
SELECT count(*) AS errors FROM Transaction WHERE status = 'Error'
```

結合條件與`AND`、`OR`和括弧：

```sql
SELECT count(*) AS slowOrFailed FROM Transaction WHERE service = 'checkout' AND (status = 'Error' OR duration > 2000)
```

針對部分符合專案使用`LIKE`搭配`%`：

```sql
SELECT count(*) AS contentRequests FROM Transaction WHERE urlPath LIKE '/content/%'
```

>[!NOTE]
>
> 字串值區分大小寫，且使用單引號： `'Error'`，而非`"Error"`或`'error'`。 以您租使用者的服務取代`checkout`。 下一個步驟將說明如何列出這些變數。

### 具有`FACET`的群組

`FACET`個群組結果，例如SQL的`GROUP BY`。 列出您的服務以及每個服務的忙碌程度：

```sql
SELECT count(*) AS requests FROM Transaction FACET service
```

多面向查詢預設會傳回前10個群組。 使用`LIMIT`檢視更多資訊：

```sql
SELECT count(*) AS requests, average(duration) AS avgMs FROM Transaction FACET service LIMIT 50
```

結合`FACET`和`TIMESERIES`以取得每個群組的一行：

```sql
SELECT count(*) AS requests FROM Transaction FACET service TIMESERIES AUTO LIMIT 5
```

在此，`LIMIT 5`保留五個最忙碌的系列。

### 將查詢儲存到儀表板

1. 開啟&#x200B;**儀表板→建立儀表板→建置自訂儀表板**，並將其命名為&#x200B;_我的第一個儀表板_。
2. 按一下&#x200B;**[新增Widget**]，選擇&#x200B;**[單一查詢]**，然後切換至&#x200B;**SQL**。
3. 貼上步驟6.7中的查詢，然後按一下&#x200B;**執行查詢**。
4. 將&#x200B;**標題**&#x200B;設定為&#x200B;_服務要求_，並選擇&#x200B;**線**&#x200B;圖表型別。
5. 按一下頂端列中的&#x200B;**儲存Widget**，然後按&#x200B;**儲存**。

您已建立您的第一個儀表板。
