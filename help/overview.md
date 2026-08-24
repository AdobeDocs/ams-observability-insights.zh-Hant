---
title: 使用 [!DNL Synoptryx]監視您的AEM Managed Services環境
description: 概略介紹Adobe [!DNL Experience Manager] Managed Services上的 [!DNL Synoptryx] 監控 — Adobe會監控哪些專案、您的帳戶設定方式，以及您的團隊如何取得存取權。
feature: Operations
role: Admin
source-git-commit: c79ae46b8ab4f6aab02821bc4446e04a94670aef
workflow-type: tm+mt
source-wordcount: '618'
ht-degree: 0%

---


# 使用[!DNL Synoptryx]監視您的AEM Managed Services環境 {#synoptryx-monitoring}

[!DNL Synoptryx]可讓您的團隊瞭解應用程式效能、基礎建設狀況以及一般使用者體驗，而不需另外設定監控平台。

>[!NOTE]
>
> [!DNL Synoptryx]產品總覽白皮書提供完整的AEM Managed Services可觀察性和監視總覽，非常適合與利害關係人共用或離線檢閱。

## 概觀 {#overview}

[!DNL Synoptryx]是Adobe的新一代可觀察性平台，其設計可提供跨應用程式效能、基礎建設狀況及綜合監控的統一可見度。 透過單一、整合的體驗，主動監控關鍵業務服務。 [!DNL Synoptryx]結合應用程式效能監控(APM)、基礎建設監控及綜合使用者歷程監控，以協助在問題影響一般使用者之前識別並解決問題。 此平台提供深層交易追蹤、JVM深入分析、基礎架構遙測和進階診斷，以加快根本原因分析速度。 它以現代可觀察性技術為基礎，提供在複雜企業環境中可擴充且安全的監控。 [!DNL Synoptryx]提供更長的資料保留、豐富的控制面板和智慧型分析，以支援卓越的營運。 透過[!DNL Adobe IMS]的順暢登入體驗，可確保安全存取與控管。 此平台旨在改善服務可靠性、加速疑難排解，並增強客戶體驗。 作為Adobe的策略可觀察性解決方案，[!DNL Synoptryx]為跨受管理服務環境的監控、自動化和營運深入分析提供可隨時因應未來的基礎。

[!DNL Synoptryx]包含在Adobe [!DNL Experience Manager] Managed Services中 — 不需要個別的監控平台或授權。 Adobe會監視您環境的可用性和效能，作為我們標準產品的一部分，而[!DNL Synoptryx]是您的團隊可用來瞭解Adobe [!DNL Experience Manager] (AEM)應用程式和支援基礎架構效能的專用平台。

本指南說明所監控的內容、您的[!DNL Synoptryx]帳戶設定方式，以及如何瀏覽您用於日常分析和疑難排解的儀表板。

## 產品一覽 {#at-a-glance}

在AEM Managed Services中，您會收到：

- **專用的[!DNL Synoptryx]帳戶** — 已布建並由Adobe Managed Services監督，您的團隊擁有唯讀存取權。
- **深層AEM交易監視** — [!DNL Synoptryx] APM代理程式會追蹤有意義交易，直到方法呼叫（包括行號）、外部相依性和存放庫作業為止。
- **統一的應用程式和基礎結構檢視** — 結合APM和主機層級的量度，以整體最佳化效能。

## 使用[!DNL Synoptryx]監視哪些Adobe {#what-we-monitor}

Adobe使用[!DNL Synoptryx] APM Java外掛程式監視AEM **作者**&#x200B;和&#x200B;**發佈**&#x200B;階層。 您拓朴中的所有託管伺服器都是透過[!DNL Synoptryx]基礎結構代理程式監視。 自訂APM和基礎結構監視在非生產和生產Managed Services環境中都啟用。

![圖表顯示跨AEM作者、發佈和託管伺服器的Synoptryx APM和基礎結構監視](assets/image6.png)

### 您帳戶中的應用程式 {#applications-in-your-account}

您的[!DNL Synoptryx]帳戶已連結至單一Adobe主帳戶，且可從多個應用程式接收資料，包括：

- 每個AEM Managed Services環境一個&#x200B;**作者**&#x200B;層級的APM應用程式
- 每個AEM Managed Services環境一個&#x200B;**發佈**&#x200B;層的APM應用程式

每個應用程式都有自己的授權金鑰。 您的Managed Services合約報表中的所有拓撲都集中到一個[!DNL Synoptryx]帳戶中。 APM和基礎結構量度和事件最多可保留&#x200B;**30天**。

## 存取許可權與您的帳戶 {#access}

監控資料已合併到Adobe布建和管理的[!DNL Synoptryx]帳戶中。 您的團隊會收到代理程式收集的所有APM和基礎結構度量的&#x200B;**完整唯讀存取權**。 Adobe Managed Services保留該帳戶的所有權和管理控制權。

>[!NOTE]
>
> **取得存取權：**&#x200B;存取[!DNL Synoptryx]需要[!DNL Adobe IMS]布建。 您的客戶成功工程師(CSE)可以為您的組織布建和管理使用者存取權。

在CSE布建帳戶後，您可以在[synoptryx.adobecqms.net](https://synoptryx.adobecqms.net)登入。

## 後續步驟 {#whats-next}

繼續使用您團隊日常使用的監控儀表板：

- [應用程式效能監視(APM)](application-performance-monitoring.md) — 追蹤AEM交易、分析JVM行為並檢查外部服務。
- [基礎架構監視](infrastructure-monitoring.md) — 檢閱主機層級的系統、網路、處理序和儲存體測量結果。
