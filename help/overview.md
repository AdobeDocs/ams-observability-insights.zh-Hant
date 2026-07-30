---
title: 使用Synoptryx監控您的AEM Managed Services環境
description: 概述Adobe Experience Manager Managed Services上的Synoptryx監控 — Adobe會監控哪些專案、您的帳戶設定方式，以及您的團隊如何取得存取權。
feature: Operations
role: Admin
source-git-commit: f937aa4e3cebd1aae6945a35a77154add5db980c
workflow-type: tm+mt
source-wordcount: '651'
ht-degree: 0%

---


# 使用Synoptryx監控您的AEM Managed Services環境 {#synoptryx-monitoring}

Synoptryx可讓您的團隊瞭解應用程式效能、基礎架構狀況及一般使用者體驗，而不需另外設定監控平台。

>[!NOTE]
>
> Synoptryx產品概觀白皮書提供完整的AEM Managed Services可觀察性和監視概觀，非常適合與利害關係人分享或離線檢閱。

## 概觀 {#overview}

Synoptryx是Adobe的新一代可觀察性平台，專為提供跨應用程式效能、基礎架構狀況及綜合監控的統一可見性而設計。 透過單一、整合的體驗，主動監控關鍵業務服務。 Synoptryx結合應用程式效能監控(APM)、基礎建設監控及綜合使用者歷程監控，協助您識別並解決問題，以免影響一般使用者。 此平台提供深層交易追蹤、JVM深入分析、基礎架構遙測和進階診斷，以加快根本原因分析速度。 它以現代可觀察性技術為基礎，提供在複雜企業環境中可擴充且安全的監控。 Synoptryx提供更長的資料保留期、豐富的儀表板和智慧型分析，以支援卓越的營運。 Adobe IMS的緊密登入體驗可確保安全存取和治理。 此平台旨在改善服務可靠性、加速疑難排解，並增強客戶體驗。 Synoptryx是Adobe的策略性可觀察性解決方案，可針對各種受管理的服務環境提供未來可使用的監控、自動化和操作深入分析基礎。

Adobe Experience Manager Managed Services隨附Synoptryx，不需個別監控平台或授權。 Adobe會監控您環境的可用性和效能，這是我們標準產品的一部分，Synoptryx是您的團隊可用來瞭解Adobe Experience Manager (AEM)應用程式和支援基礎架構效能的專用平台。

本指南說明所監控的內容、您的Synoptryx帳戶設定方式，以及如何導覽您用於日常分析和疑難排解的儀表板。

## 產品一覽 {#at-a-glance}

在AEM Managed Services中，您會收到：

- **專用的Synoptryx帳戶** — 已布建並由Adobe Managed Services監督，您的團隊擁有唯讀存取權。
- **深層AEM交易監視** — Synoptryx APM代理程式會追蹤有意義交易，一直到方法呼叫（包括行號）、外部相依性和存放庫作業。
- **統一的應用程式和基礎結構檢視** — 結合APM和主機層級的量度，以整體最佳化效能。

## 使用Synoptryx監控什麼Adobe {#what-we-monitor}

Adobe使用Synoptryx APM Java外掛程式監視AEM **作者**&#x200B;和&#x200B;**發佈**&#x200B;階層。 您拓撲中的所有託管伺服器都會使用Synoptryx基礎結構代理程式進行監視。 自訂APM和基礎結構監視在非生產和生產Managed Services環境中都啟用。

![圖表顯示跨AEM作者、發佈和託管伺服器的Synoptryx APM和基礎結構監視](assets/image6.png)

### 您帳戶中的應用程式 {#applications-in-your-account}

您的Synoptryx帳戶已連結至單一Adobe主帳戶，並可接收來自多個應用程式的資料，包括：

- 每個AEM Managed Services環境一個&#x200B;**作者**&#x200B;層級的APM應用程式
- 每個AEM Managed Services環境一個&#x200B;**發佈**&#x200B;層的APM應用程式

每個應用程式都有自己的授權金鑰。 Managed Services合約中的所有拓撲都會報告至一個Synoptryx帳戶。 APM和基礎結構量度和事件最多可保留&#x200B;**30天**。

## 存取許可權與您的帳戶 {#access}

監控資料會整合至Adobe布建和管理的Synoptryx帳戶中。 您的團隊會收到代理程式收集的所有APM和基礎結構度量的&#x200B;**完整唯讀存取權**。 Adobe Managed Services保留該帳戶的所有權和管理控制權。

>[!NOTE]
>
> **存取許可權：**&#x200B;存取Synoptryx需要Adobe IMS布建。 您的客戶成功工程師(CSE)可以為您的組織布建和管理使用者存取權。

在CSE布建帳戶後，您可以在[synoptryx.adobecqms.net](https://synoptryx.adobecqms.net)登入。

## 後續步驟 {#whats-next}

繼續使用您團隊日常使用的監控儀表板：

- [應用程式效能監視(APM)](application-performance-monitoring.md) — 追蹤AEM交易、分析JVM行為並檢查外部服務。
- [基礎架構監視](infrastructure-monitoring.md) — 檢閱主機層級的系統、網路、處理序和儲存體測量結果。
