---
title: 使用可觀察性深入分析監控您的AEM Managed Services環境
description: 從這裡開始瞭解AEM Managed Services中的「可觀察性深入分析」包含哪些內容、其服務對象，以及如何導覽本指南的其餘部分。
feature: Operations
role: Admin
source-git-commit: 94ba857f5b6a5c33483e4d49f5a1daa9583b6347
workflow-type: tm+mt
source-wordcount: '737'
ht-degree: 0%

---


# 使用可觀察性深入分析監控您的AEM Managed Services環境 {#observability-insights-monitoring}

「可觀察性深入分析」可讓您在Adobe Experience Manager Managed Services中檢視應用程式效能、基礎架構健全狀況和服務行為，而不需要個別的監控平台。

如果您負責服務可靠性、事件回應或效能分析，可觀察性深入分析可協助您快速從症狀移至證據。 它結合應用程式遙測和主機層級的健全狀況訊號，因此客戶團隊和Adobe Managed Services可以從共用作業檢視中調查問題。

## 團隊為何要使用可觀察性深入分析？ {#why-teams-use-observability-insights}

使用「可觀察性深入分析」來回答操作問題，例如：

- 問題是否會影響「作者」、「發佈」，或同時影響兩者？
- 問題是否由應用程式行為、主機資源壓力或兩者的組合所造成？
- 哪些交易、端點或狀態群組可解釋錯誤或延遲的尖峰？
- 問題是否孤立於一個環境，或是橫跨更廣大的拓撲結構皆可見？

「可觀察性深入分析」是針對最近行為的作業分析而設計。 它可協助您識別哪些專案已變更、變更的位置，以及哪些訊號在升級或修正動作之前最相關。

## 什麼可觀察性深入解析可協助您這麼做？ {#what-observability-insights-helps-you-do}

使用可觀察性深入分析來：

- 瞭解製作和發佈層級在實際流量下的行為。
- 將應用程式延遲、錯誤率和JVM健康情況與主機層級訊號建立關聯。
- 確認問題孤立於一個環境、一個層級或一個主機。
- 在調查期間為Adobe Managed Services和您的內部團隊提供共用的運作檢視。

AEM Managed Services隨附可觀察性深入分析。 Adobe會布建和管理帳戶、工具支援的環境，並將產生的儀表板以唯讀操作工具的形式向您的團隊公開。

由於Adobe會管理平台設定和檢測，因此您可以專注於調查和解釋，而不是代理程式部署、帳戶管理或儀表板元件。

## 產品一覽 {#at-a-glance}

在AEM Managed Services中，您會收到：

- **專用可觀察性深入分析帳戶** — 由Adobe Managed Services布建和監督，為您的團隊提供唯讀存取權。
- **深層AEM交易監視** — Observability Insights APM代理程式會追蹤有意義交易，一直到方法呼叫（包括行號）、外部相依性和存放庫作業。
- **整合式應用程式和主機檢視** — 結合應用程式和主機層級量度，以整體最佳化效能。

## 本檔案的適用對象 {#who-this-documentation-is-for}

本檔案主要針對：

- 需要深入瞭解受監控環境的AEM Managed Services管理員
- 處理事件、趨勢分析和服務審查的運作和支援團隊
- 客戶工程團隊在調查期間與Adobe Managed Services合作
- 需要瞭解監控範圍和營運責任的利害關係人

## Adobe使用可觀察性深入分析監控什麼 {#what-we-monitor}

Adobe使用Observability Insights APM Java外掛程式監視AEM **作者**&#x200B;和&#x200B;**發佈**&#x200B;階層。 您拓朴中的所有託管伺服器都會使用Observability Insights Infrastructure代理程式進行監視。 自訂APM和基礎結構監視在非生產和生產Managed Services環境中都啟用。

![圖表顯示跨AEM作者、發佈和託管伺服器的Observability Insights APM和基礎結構監視](v2-assets/login-screen.png)

### 您帳戶中的應用程式 {#applications-in-your-account}

您的Observability Insights帳戶已連結至單一Adobe主帳戶，並可接收來自多個應用程式的資料，包括：

- 每個AEM Managed Services環境一個&#x200B;**作者**&#x200B;層級的APM應用程式
- 每個AEM Managed Services環境一個&#x200B;**發佈**&#x200B;層的APM應用程式

每個應用程式都有自己的授權金鑰。 Managed Services合約中的所有拓撲都會報告至一個「可觀察性深入分析」帳戶。 APM和基礎結構量度和事件最多可保留&#x200B;**30天**。

## 存取您的帳戶 {#access}

監控資料會整合至Adobe布建和管理的「可觀察性深入分析」帳戶中。 客戶使用者會收到代理程式所收集之APM和基礎結構資料的&#x200B;**唯讀存取權**。 Adobe Managed Services保留帳戶擁有權和管理控制權。

### 先決條件 {#access-prerequisites}

登入前，請先確認下列事項：

- 您的組織擁有使用中的&#x200B;**AEM Managed Services**&#x200B;訂閱。 其中不包含可觀察性深入分析，不需額外付費。
- 您的客戶成功工程師(CSE)已布建您的Adobe IMS帳戶，並授予您組織可觀察性深入分析帳戶的存取權。

>[!NOTE]
>
> **若要取得存取權：**&#x200B;若要存取Observability Insights，必須使用Adobe IMS布建。 請聯絡您的客戶成功工程師(CSE)，以布建和管理組織的使用者存取權。

CSE布建帳戶後，請登入[insights.adobecqms.net](https://insights.adobecqms.net)。 此URL對所有AEM Managed Services客戶而言都相同；您組織的環境和儀表板屬於您布建的帳戶的範圍。
