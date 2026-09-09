---
source-git-commit: e5523081fcd68500602e5d1bf853694d1f6c3980
workflow-type: tm+mt
source-wordcount: '1104'
ht-degree: 7%

---
# 可觀察性深入分析公用API

可觀察性深入分析公用API可讓您直接將自己的可觀察性資料（要求概述、服務目錄、追蹤和量度）提取到您自己的工具、指令碼和儀表板中。

- **API基底URL (API_BASE_URL)：** `https://insights.adobecqms.net/`
- **格式：** HTTPS上的JSON
- **驗證：** API金鑰（持有人權杖）

> 以您的Observability Insights執行個體的API主機（例如`https://insights.adobecqms.net/`）取代整個檔案的`{{API_BASE_URL}}`。

&#x200B;---

## &#x200B;1. 取得API金鑰

API金鑰是繫結至您帳戶的個人認證，其範圍設定為單一組織。 索引鍵只能讀取屬於建立該索引鍵的組織之租使用者的資料，永遠無法看到其他組織的資料。

### 產生金鑰

1. 登入[可觀察性深入分析儀表板](https://insights.adobecqms.net/)。
2. 開啟&#x200B;**API金鑰**→設定檔功能表（右上方）。
   ![API金鑰功能表](v2-assets/api-key.png)
3. 在&#x200B;**API金鑰**&#x200B;索引標籤中，按一下&#x200B;**產生金鑰**。
   ![產生API金鑰](v2-assets/api-key-gen.png)
4. 為其指定描述性名稱（例如`CI pipeline`、`Grafana datasource`）、選擇其範圍應設的組織，並選擇性地設定到期日。
5. 按一下&#x200B;**產生金鑰**。 您的金鑰會以下列格式顯示&#x200B;**一次**：

   ```
   synx_9pQ2v6f1WYbLZk3n0aRtEo4jXcHsVmDgUiPq7B8l1yc
   ```

   **立即複製並儲存在安全的位置** （機密管理員、CI機密存放區等）  — 儀表板無法再次向您顯示。 如果遺失它，請撤銷它並產生新的它。

### 管理現有金鑰

API金鑰區段會列出您建立的每個金鑰，包括其組織、建立日期、有效期和上次使用的時間戳記。 按一下&#x200B;**撤銷**&#x200B;索引鍵旁的垃圾桶圖示 — 撤銷立即且無法復原。

### 金鑰安全性

- 將API金鑰視為與密碼完全相同的密碼。 擁有金鑰的任何人都可以讀取其所屬組織內每個租使用者的所有可觀察資料，直到資料被撤銷或過期為止。
- 切勿將金鑰提交至原始檔控制或以純文字（聊天、電子郵件、票證）共用。
- 定期輪換金鑰並撤銷任何不再使用的金鑰。
- 如果金鑰遭到破壞，請立即從&#x200B;**組織設定→API金鑰**&#x200B;中撤銷該金鑰，並產生替代金鑰。

&#x200B;---

## &#x200B;2. 驗證請求

對公開API的每個請求都必須在`Authorization`標頭中包含您的金鑰：

```
Authorization: Bearer synx_9pQ2v6f1WYbLZk3n0aRtEo4jXcHsVmDgUiPq7B8l1yc
```

沒有有效金鑰或金鑰過期/撤銷的請求會收到`401 Unauthorized`。 工作階段登入（瀏覽器Cookie/權杖）在此API上&#x200B;**不接受**。

&#x200B;---

## &#x200B;3. 基本概念

### 租用戶

每個端點都需要識別要讀取哪個租使用者資料的`tenant_id`查詢引數。 索引鍵只能查詢屬於建立它的組織的租使用者；若要求該組織以外的租使用者，則會傳回`403 Forbidden`。 此API上沒有「所有租使用者」模式 — 一律傳遞特定`tenant_id`。

不確定您的金鑰可以使用哪些`tenant_id`值？ 呼叫[`GET /public/v1/tenants`](#get-publicv1tenants) — 它列出您金鑰有權查詢的租使用者。

### 時間範圍

接受`from` / `to`引數的端點會採用Unix時間戳記（秒）、毫秒時間戳記或ISO 8601日期時間字串，例如：

```
from=1735689600
from=2025-01-01T00:00:00Z
```

如果省略，大部分端點會預設為最近的滾動時段（請參閱下面的每個端點）。

### 速率限制

請求是每個API金鑰限定的速率。 如果超過限制，您將收到：

```http
HTTP/1.1 429 Too Many Requests
Retry-After: 60

{ "error": "Too Many Requests", "message": "Rate limit of 300 requests/60s exceeded" }
```

在`Retry-After`標頭中的秒數後關閉並重試。 如果您的使用案例需要更高的限制，請聯絡支援人員。

### 錯誤次數

錯誤會以JSON格式傳回，其中包含`error`欄位，且通常是人類看得懂的`message`：

```json
{ "error": "Bad Request", "message": "tenant_id is required" }
```

| 狀態 | 含義 |
| ------------------------- | ------------------------------------------------------------------ |
| `400 Bad Request` | 遺失或無效的引數（例如無`tenant_id`、錯誤的時間範圍） |
| `401 Unauthorized` | API金鑰遺失、無效、過期或撤銷 |
| `403 Forbidden` | 要求的租使用者未授權此金鑰 |
| `429 Too Many Requests` | 超過速率限制 — 請參閱`Retry-After` |
| `502 Bad Gateway` | 上游查詢失敗 — 可安全重試 |
| `503 Service Unavailable` | 資料後端暫時無法使用 |

&#x200B;---

## &#x200B;4. 端點

### `GET /public/v1/tenants`

列出您的金鑰有權查詢的租使用者ID。 請先呼叫此專案 — 其他每個端點都需要其中一個值做為`tenant_id`。

```bash
curl -s "{{API_BASE_URL}}/public/v1/tenants" \
  -H "Authorization: Bearer $Observability_Insights_API_KEY"
```

```json
{ "tenants": ["tenant1", "tenant2"] }
```

### `GET /public/v1/overview`

租使用者在一段時間內的高階健康情況KPI：請求量、錯誤率和延遲百分位數。

| 引數 | 必要 | 說明 |
| ------------ | -------- | ------------------------------------------------------------------------- |
| `tenant_id` | 是 | 要查詢的租使用者 |
| `from`, `to` | 否 | 時間範圍（請參閱[時間範圍](#time-ranges)） |
| `minutes` | 否 | 如果未指定`from`/`to` （預設為`15`），則為「最後N分鐘」的簡稱 |

```bash
curl -s "{{API_BASE_URL}}/public/v1/overview?tenant_id=<tenant_id>&minutes=30" \
  -H "Authorization: Bearer $Observability_Insights_API_KEY"
```

```json
{
  "tenant_id": "<tenant_id>",
  "from": 1735689000,
  "to": 1735690800,
  "total_spans": 48213,
  "errors": 112,
  "error_rate_pct": 0.23,
  "p50_ms": 34,
  "p95_ms": 210,
  "p99_ms": 480,
  "service_count": 12,
  "trace_count": 9021
}
```

### `GET /public/v1/services`

列出租使用者的不同服務名稱報告。

| 引數 | 必要 | 說明 |
| ------------ | -------- | --------------------------------------------------------------------- |
| `tenant_id` | 是 | 要查詢的租使用者 |
| `from`, `to` | 否 | 限制在此視窗中看到的服務；預設為過去7天 |

```bash
curl -s "{{API_BASE_URL}}/public/v1/services?tenant_id=<tenant_id>" \
  -H "Authorization: Bearer $Observability_Insights_API_KEY"
```

```json
{
  "tenant_id": "<tenant_id>",
  "services": ["checkout-api", "payments-worker", "web-frontend"]
}
```

### `GET /public/v1/traces`

使用選用的篩選器搜尋租使用者的最近追蹤。

| 引數 | 必要 | 說明 |
| ----------------- | -------- | ------------------------------------------------- |
| `tenant_id` | 是 | 要查詢的租使用者 |
| `from`, `to` | 否 | 時間範圍；預設為過去24小時 |
| `limit` | 否 | 傳回的最大列數（1-200、預設100） |
| `offset` | 否 | 分頁位移（預設為0） |
| `service` | 否 | 依服務名稱篩選 |
| `app_name` | 否 | 依應用程式/執行個體名稱篩選 |
| `status` | 否 | 依追蹤狀態篩選： `ok`、`error`或`unset` |
| `search` | 否 | 跨範圍/作業名稱的任意文字搜尋 |
| `min_duration_ms` | 否 | 僅追蹤超過此持續時間的專案 |

```bash
curl -s "{{API_BASE_URL}}/public/v1/traces?tenant_id=<tenant_id>&status=error&limit=25" \
  -H "Authorization: Bearer $Observability_Insights_API_KEY"
```

```json
{
  "data": [
    {
      "TraceId": "4bf92f3577b34da6a3ce929d0e0e4736",
      "ServiceName": "checkout-api",
      "DurationMs": 812,
      "StatusCode": "Error",
      "Timestamp": "2026-08-30T09:12:44Z"
    }
  ],
  "rows": 137,
  "limit": 25,
  "offset": 0
}
```

使用`rows` （總比對計數）與`limit`/`offset`一起翻閱結果。

### `GET /public/v1/traces/:traceId`

傳回單一追蹤的完整跨度瀑布。

| 引數 | 必要 | 說明 |
| ----------- | -------- | ---------------------------------------- |
| `tenant_id` | 是 | 追蹤所屬的租使用者 |
| `limit` | 否 | 要傳回的最大跨距（1-500，預設500） |
| `offset` | 否 | 非常大型追蹤的分頁位移 |

```bash
curl -s "{{API_BASE_URL}}/public/v1/traces/4bf92f3577b34da6a3ce929d0e0e4736?tenant_id=<tenant_id>" \
  -H "Authorization: Bearer $Observability_Insights_API_KEY"
```

```json
{
  "spans": [
    {
      "SpanId": "00f067aa0ba902b7",
      "Name": "POST /checkout",
      "DurationMs": 812,
      "children": []
    }
  ],
  "totalDurationMs": 812,
  "spanCount": 14,
  "limit": 500,
  "offset": 0
}
```

### `GET /public/v1/metrics`

傳回租使用者的原始量度資料點。

| 引數 | 必要 | 說明 |
| ---------------------------------- | ---------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------- |
| `tenant_id` | 是 | 要查詢的租使用者 |
| `metric` | `metric`/`like`之一 | 精確的量度名稱 |
| `like` | `metric`/`like`之一 | SQL `LIKE`模式以符合多個量度名稱 |
| `type` | 否 | `gauge` （預設）或`sum` |
| `from`, `to` | 否 | 時間範圍；預設為過去24小時 |
| `service` | 否 | 依服務名稱篩選 |
| `host` | 否 | 依主機名稱篩選。 下列基礎架構主機量度需要 — 若沒有它，租使用者中每個主機的讀取會混合在一起 |
| `attribute_key`, `attribute_value` | 否 | 依特定量度屬性篩選（必須搭配使用） |

```bash
curl -s "{{API_BASE_URL}}/public/v1/metrics?tenant_id=<tenant_id>&metric=jvm.memory.used&type=gauge" \
  -H "Authorization: Bearer $Observability_Insights_API_KEY"
```

```json
{
  "data": [
    {
      "TimeUnix": "2026-08-30T09:00:00Z",
      "MetricName": "jvm.memory.used",
      "Value": 512482816,
      "ServiceName": "checkout-api",
      "host": ""
    }
  ],
  "rows": 1
}
```

#### 基礎結構主機測量結果

相同的端點也會提供基礎架構儀表板（CPU、記憶體、平均負載、磁碟I/O、網路I/O）上顯示的主機層級量度。 使用這些確切的`metric` / `attribute_key` / `attribute_value`組合，一律使用`host`：

| 儀表板Widget | `metric` | `attribute_key` | `attribute_value` |
| --------------------- | ------------------------------------- | --------------- | ------------------------------------------------------------------------------------------- |
| CPU % | `system.cpu.utilization` | `state` | `idle` （從1減去「使用中」），或分別查詢`user`/`system`/`iowait`並加總 |
| 記憶體使用率% | `system.memory.utilization` | `state` | `used` |
| 平均載入（1分鐘） | `system.cpu.load_average.1m` | — | — |
| 磁碟讀取I/O | `system.disk.io` (`type=sum`) | `direction` | `read` |
| 磁碟寫入I/O | `system.disk.io` (`type=sum`) | `direction` | `write` |
| 磁碟讀取作業 | `system.disk.operations` (`type=sum`) | `direction` | `read` |
| 磁碟寫入作業 | `system.disk.operations` (`type=sum`) | `direction` | `write` |
| 中的網路 | `system.network.io` (`type=sum`) | `direction` | `receive` |
| 網路輸出 | `system.network.io` (`type=sum`) | `direction` | `transmit` |

```bash
curl -s "{{API_BASE_URL}}/public/v1/metrics?tenant_id=<tenant_id>&metric=system.cpu.utilization&type=gauge&attribute_key=state&attribute_value=idle&host=<host_name>" \
  -H "Authorization: Bearer $Observability_Insights_API_KEY"
```

**重要 — 磁碟和網路值是原始值，不斷增加的計數器，而不是速率。** 控制面板的「位元組/秒」和「作業/秒」圖表是透過兩個連續的計數器讀數除以經過的時間來計算的：

```
rate = (value_at_t2 - value_at_t1) / (t2 - t1_in_seconds)
```

### `GET /public/v1/pages`

每個Dispatcher執行個體的熱門請求內容頁面(`.html`)，依請求計數排名。 由`dispatcher.httpd.requests`量度支援 — 此端點專屬於AEM Dispatcher/CDN樣式的存取記錄，而不是一般的頁面分析工具。

| 引數 | 必要 | 說明 |
| ------------ | -------- | -------------------------------------- |
| `tenant_id` | 是 | 要查詢的租使用者 |
| `from`, `to` | 否 | 時間範圍；預設為過去24小時 |
| `limit` | 否 | 傳回的最大列數（1-500、預設50） |

```bash
curl -s "{{API_BASE_URL}}/public/v1/pages?tenant_id=<tenant_id>&limit=50" \
  -H "Authorization: Bearer $Observability_Insights_API_KEY"
```

```json
{
  "tenant_id": "<tenant_id>",
  "from": 1735689000,
  "to": 1735690800,
  "data": [
    {
      "instance": "<instance_name>",
      "domain": "www.abc.com",
      "path": "/join-us/insights.html",
      "full_url": "https://www.abc.com/join-us/insights.html",
      "requests": 7
    }
  ],
  "rows": 1
}
```

&#x200B;---

## &#x200B;5. 此API沒有的作用

- **沒有原始SQL存取權。** 所有端點都會傳回已組織且專門建置的資料圖形 — 您無法直接查詢基礎資料存放區。
- **沒有跨租使用者查詢。** 每個請求的範圍剛好是一個`tenant_id`。
- **沒有寫入許可權。** 公用API是唯讀的。

&#x200B;---

## &#x200B;6. 支援

如果您遇到非預期的錯誤，或這些端點未涵蓋的使用案例，請聯絡您的客戶成功/啟用工程師以取得進一步協助。
