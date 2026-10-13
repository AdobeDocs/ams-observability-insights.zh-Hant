---
title: 附錄
description: 參考可觀察性深入分析SQL查詢和儀表板的補充資訊。
source-git-commit: 143a9567da06f0137ef690795cfbc41bda1780ca
workflow-type: tm+mt
source-wordcount: '145'
ht-degree: 5%
---

## 附錄 {#appendix}

**在此檔案中：** [14.1限制](#141-limits) · [14.2匯出檔案格式](#142-export-file-format) · [14.3單頁速查表](#143-one-page-cheat-sheet)

### 限制

| 項目 | 限制 |
| -------------------------------- | ----------------------------------------------------------------------- |
| 查詢文字 | 4 KB |
| `FACET`屬性 | 5 |
| `LIMIT` | 1 - 5 000 （預設值：數字1、Facet 10、事件100、時間序列5 000） |
| `IN (…)`個值 | 1 - 100 |
| 時間視窗 | 31天 |
| 查詢逾時 | 15秒（加上位元組讀取預算） |
| `TIMESERIES AUTO`最小貯體 | 2 分鐘 |
| 公式 | 2 - 6個查詢；運算式≤ 500個字元；名稱≤ 40個字元 |
| 儀表板 | ≤ 40個標籤·每個標籤≤ 200個Widget · ≤ 50個變數 |
| 小工具 | title ≤ 200 ·子標題≤ 500 ·單位≤ 40 · ≤ 10臨界值 |
| 文字Widget | ≤ 4 000個字元 |
| 變數 | 名稱≤ 64個字元· ≤ 500個選項/值·查詢≤ 4 KB |

### 匯出檔案格式

使用&#x200B;**匯出**&#x200B;產生此專案；**匯入**&#x200B;載入此專案。 對於版本控制儀表板或在租使用者之間促銷儀表板很有用。

```json
{
  "name": "Golden Signals",
  "templateVariables": [
    {
      "id": "var-1",
      "name": "svc",
      "label": "Service",
      "kind": "query",
      "multi": true,
      "query": "SELECT count(*) FROM Transaction FACET service LIMIT 200",
      "value": null
    }
  ],
  "pages": [
    {
      "id": "page-1",
      "name": "Overview",
      "widgets": [
        {
          "id": "sec-kpis",
          "title": "SLO KPIs",
          "queryLanguage": "text",
          "text": "",
          "isSection": true,
          "collapsed": false,
          "layout": { "x": 0, "y": 0, "w": 12, "h": 1 }
        },
        {
          "id": "w-error-rate",
          "title": "Error %",
          "queryLanguage": "SQL",
          "query": "SELECT percentage(*, WHERE status = 'Error') AS errorRate FROM Transaction",
          "vizType": "stat",
          "unit": "%",
          "thresholds": [
            { "value": 0, "status": "ok" },
            { "value": 1, "status": "warning" },
            { "value": 5, "status": "error" }
          ],
          "layout": { "x": 0, "y": 1, "w": 3, "h": 2 }
        },
        {
          "id": "w-throughput",
          "title": "Throughput by service",
          "queryLanguage": "SQL",
          "query": "SELECT count(*) AS requests FROM Transaction FACET service TIMESERIES AUTO LIMIT 8",
          "vizType": "area",
          "layout": { "x": 3, "y": 1, "w": 9, "h": 4 }
        },
        {
          "id": "w-availability",
          "title": "Availability %",
          "queryLanguage": "SQL-formula",
          "formulaQueries": [
            {
              "name": "ok",
              "query": "SELECT count(*) AS v FROM Transaction WHERE status != 'Error'"
            },
            {
              "name": "total",
              "query": "SELECT count(*) AS v FROM Transaction"
            }
          ],
          "formulaExpression": "(ok / total) * 100",
          "vizType": "stat",
          "unit": "%",
          "layout": { "x": 0, "y": 3, "w": 3, "h": 2 }
        }
      ]
    }
  ],
  "exportedAt": "2026-10-01T00:00:00.000Z"
}
```

`vizType`值： `timeseries` （行）、`area`、`bar`、`pie`、`stat` （數字）、`table`。 閾值`status`： `ok` （綠色）、`warning` （琥珀色）、`error` （紅色）。

### 單頁速查表

```text
SHAPE        SELECT agg FROM Entity                         -> Number
             ... FACET a[, b..e] LIMIT n                     -> Bar / Pie / Table
             ... TIMESERIES AUTO                             -> Line / Area
             ... FACET a TIMESERIES AUTO LIMIT n             -> top-n series
             SELECT attr1, attr2 FROM Entity LIMIT n         -> event Table

AGGREGATES   count(*)  average(x)  sum(x)  min(x)  max(x)  uniqueCount(x)
             percentile(x, 95)  stddev(x)  latest(x[, rank])  earliest(x[, rank])
             percentage(*, WHERE c)  apdex(duration, 500)  percentOfTotal(x)
             rate(value)        -- counters, TIMESERIES only
             derivative(value)  -- gauges,   TIMESERIES only
             agg FILTER (WHERE c) AS alias

WHERE        = != < <= > >=   LIKE / NOT LIKE ('%', '_')   IN / NOT IN (...)
             IS [NOT] NULL   AND OR NOT ( )   strings in 'single quotes'
             string attr present?  ->  attr != ''

ORDER/LIMIT  ORDER BY alias [ASC|DESC]   LIMIT 1..5000

VARIABLES    = {{name}}   IN ({{name}})   numeric: > {{name}}

FORMULA      2-6 named queries, one metric each, no FACET, same shape
             expression: + - * / ( ) numbers names
```
