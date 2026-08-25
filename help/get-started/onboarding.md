---
title: 開始使用可觀察性深入分析
description: 瞭解如何存取Observability Insights、Adobe代表您監控哪些專案，以及在本指南中哪裡可以找到您需要的內容。
feature: Operations
role: Admin
source-git-commit: cc405e8b70973c33ecc6137114315998e8f9af50
workflow-type: tm+mt
source-wordcount: '533'
ht-degree: 0%

---


# 開始使用可觀察性深入分析 {#get-started}

本節說明新使用者的基本知識：如何存取您的Observability Insights帳戶、Adobe會代表您監控哪些環境和資料，以及如何導覽本檔案的其餘部分。

## 可觀察性深入解析介面 {#observability-insights-interface}

當您在[insights.adobecqms.net](https://insights.adobecqms.net)登入時，開啟的畫面會提供進入您的AEM Managed Services環境之所有監控區域的進入點。

![可觀察性深入分析開啟熒幕，顯示APM和基礎結構監視進入點](../v2-assets/observability-catalog-listing.png)

介面會依兩個核心監控區域組織：

- **應用程式** — 顯示作者與發佈層級的應用程式效能資料。 使用此專案來調查要求輸送量、錯誤率、延遲、JVM行為和追蹤層級執行詳細資訊。 檢視[應用程式](../applications.md)。
- **主機** — 顯示您受管理拓撲的主機層級健康情況資料。 使用此功能來評估個別伺服器上的CPU、記憶體、磁碟、網路和儲存訊號。 請參閱[主機](../hosts.md)。

這兩個區域對於客戶使用者都是唯讀的。 Adobe Managed Services會管理帳戶布建、檢測和管理控制。

## 存取與帳戶管理 {#access-overview}

可觀察性深入分析存取是透過Adobe IMS管理的。 Adobe會布建和管理您組織的帳戶；客戶團隊會獲得所有受監控資料的唯讀存取權。

要點：

- 您組織的「可觀察性深入分析」帳戶已連結至單一Adobe主帳戶。
- 您的Managed Services合約中的所有環境（製作和發佈、生產和非生產）都會報告至此帳戶。
- 使用者存取權由您的客戶成功工程師(CSE)布建和管理。

如需布建步驟、使用者角色，以及客戶使用者可以和不能執行的動作，請參閱[存取和帳戶管理](access-and-accounts.md)。

## 涵蓋範圍、環境和資料保留 {#coverage-overview}

Adobe使用Observability Insights APM Java外掛程式來監控AEM的製作和發佈層級，並使用Observability Insights Infrastructure代理程式監控所有託管伺服器。 監控功能會在非生產環境和生產環境中啟用。

要點：

- 每個AEM Managed Services環境都包含一個APM應用程式，分別適用於「作者」和「發佈」。
- APM量度、基礎結構量度和事件最多可保留&#x200B;**30天**。
- 「可觀察性深入分析」適合用於營運分析和最近的趨勢比較；它不是封存或長期報告工具。 在資料過期之前擷取熒幕擷取畫面或匯出的證據。

如需完整涵蓋範圍的詳細資訊，包括應用程式在您的帳戶中的呈現方式以及保留期間的營運影響，請參閱[涵蓋範圍、環境和資料保留](coverage-and-data.md)。

## 如何建構本指南 {#how-this-guide-is-structured}

本檔案分為四個區域。 使用下列說明直接前往您需要的。

**開始使用** — 此節。 涵蓋存取、帳戶布建、監控範圍和資料保留。

**[使用可觀察性深入分析](../use-observability-insights.md)** — 以任務為導向的日常調查指引。 當症狀是面向應用程式時，請使用[應用程式](../applications.md)：頁面速度慢、錯誤尖峰或不穩定的交易。 當您需要判斷主機層級的資源壓力（CPU、記憶體、磁碟或網路）是否說明您在應用程式中看到的內容時，請使用[主機](../hosts.md)。 在[調查應用程式問題](../use-cases/investigate-application-issues.md)和[調查基礎結構問題](../use-cases/investigate-infrastructure-issues.md)中可以使用逐步調查流程。

**[常見問題](../troubleshooting/common-questions.md)** — 當您不確定從何處開始或在使用中事件期間需要快速解答時，的常見問題和支援導向的進入點。
