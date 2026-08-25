---
title: 涵蓋範圍、環境和資料保留
description: 檢視AEM Managed Services中的「可觀察性深入分析」監控了哪些專案、應用程式的呈現方式，以及監控資料會保留多長時間。
feature: Operations
role: Admin
source-git-commit: 1d54a6a398360b040221db5b2780d301722894bf
workflow-type: tm+mt
source-wordcount: '267'
ht-degree: 1%

---


# 涵蓋範圍、環境和資料保留 {#coverage-environments-and-data-retention}

此頁面概述在AEM Managed Services的可觀察性深入分析中收集哪些資料，以及該資料的組織方式。

## 監視涵蓋範圍 {#monitoring-coverage}

Adobe監視器：

- 使用Observability Insights APM Java外掛程式的AEM作者階層
- 使用Observability Insights APM Java外掛程式的AEM發佈階層
- 使用Observability Insights Infrastructure代理程式，在受管理的拓撲中託管伺服器

自訂APM和基礎結構監視在非生產和生產Managed Services環境中都啟用。

## 應用程式的呈現方式 {#how-applications-are-represented}

每個AEM Managed Services環境通常包括：

- 一個APM應用程式供作者使用
- 一個用於發佈的APM應用程式

Managed Services合約中的所有拓撲都會報告至一個「可觀察性深入分析」帳戶。

## 資料保留 {#data-retention}

APM量度、基礎結構量度和相關事件最多可保留&#x200B;**30天**。

## 摘要表格 {#summary-tables}

| 涵蓋範圍 | 監控的內容 |
| -------------- | ------------------------------------------ |
| APM | AEM製作和發佈應用程式 |
| 基礎架構 | 受管理拓撲中的所有託管伺服器 |

| 項目 | 表示 |
| ------------------------------ | ------------------------------------------------------------- |
| AEM環境 | 一個作者APM應用程式和一個發佈APM應用程式 |
| Observability Insights帳戶 | 每個Managed Services客戶範圍一個Adobe管理帳戶 |

| 資料類型 | 保留 |
| --------------------------------- | ------------- |
| APM量度和事件 | 最多30天 |
| 基礎架構量度和事件 | 最多30天 |

## 這在操作上代表什麼意義 {#what-this-means-operationally}

- 「可觀察性深入分析」適用於營運分析、作用中事件及最近的趨勢比較。
- 如有需要，保留期間以外的歷史分析應透過其他報告或封存程式來處理。
- 在調查週期性問題時，在資料過期之前擷取熒幕擷取畫面或匯出的證據。
