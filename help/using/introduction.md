---
title: 简介 [!DNL Asset Compute Service]
description: '"[!DNL Asset Compute Service] 是一种云原生资产处理服务，可降低复杂性并提高可扩展性。”'
exl-id: f8c89f65-5a94-44f3-aaac-4612ae291101
source-git-commit: c6f747ebd6d1b17834f1af0837609a148804f8a9
workflow-type: tm+mt
source-wordcount: '278'
ht-degree: 1%

---

# 概述 [!DNL Asset Compute Service] {#overview}

[!DNL Asset Compute Service] 是一项可扩展和可扩展的服务，其 [!DNL Adobe Experience Cloud] 处理数字资产。 它可以将图像、视频、文档和其他文件格式转换为不同的呈现形式，包括缩略图、提取的文本和元数据以及存档。

开发人员可以插入自定义资产应用程序（也称为自定义工作程序），以解决自定义用例。 该服务在Adobe上工作 [!DNL I/O Runtime]. 它可通过以下方式扩展 [!DNL Adobe Developer App Builder] 使用Node.js编写的Headless应用程序。 他们可以执行自定义操作，例如调用外部API来执行图像操作或利用 [!DNL Adobe Sensei] 支持。

[!DNL Adobe Developer App Builder] 是一个在Adobe上构建和部署自定义Web应用程序的框架 [!DNL I/O Runtime] 以扩展Adobe Experience Cloud解决方案。 要创建自定义应用程序，开发人员可以利用 [!DNL React Spectrum] (Adobe的UI工具包)、创建微服务、创建自定义事件和编排API。 请参阅 [Adobe Developer App Builder文档](https://developer.adobe.com/app-builder/docs/overview/).

>[!NOTE]
>
>目前， [!DNL Asset Compute Service] 只能通过以下方式使用 [!DNL Experience Manager] as a [!DNL Cloud Service]. 管理员创建可调用 [!DNL Asset Compute Service] 以传递要处理的资产。 请参阅 [使用资产微服务和处理配置文件](https://experienceleague.adobe.com/zh-hans/docs/experience-manager-cloud-service/content/assets/manage/asset-microservices-configure-and-use).

## 支持的用例 [!DNL Asset Compute Service] {#possible-use-cases-benefits}

[!DNL Asset Compute Service] 支持一些常见的业务用例，如基本图像处理、Adobe应用程序特定的转换以及协调复杂业务需求的自定义应用程序创建。

您可以使用 [!DNL Asset Compute] 用于为不同文件类型生成缩略图的Web服务，高质量图像渲染 [支持的文件格式](https://experienceleague.adobe.com/en/docs/experience-manager-cloud-service/content/assets/file-format-support). 请参阅 [通过自定义配置支持的用例](https://experienceleague.adobe.com/zh-hans/docs/experience-manager-cloud-service/content/assets/manage/asset-microservices-configure-and-use).

>[!NOTE]
>
>该服务不提供资产存储。 用户提供它并对云存储中的源文件和演绎版文件位置提供引用。

<!-- TBD: Should this be mentioned in the docs?

|Asset Compute Service does not do this|Expectations from implementing client|
|---|---|
| Binary uploads or API-based asset ingestion. | Use other methods to ingest assets. |
| Store binaries or any persisted data across processing requests.| Each request is independent so treat it as a standalone request by sharing binary and processing instructions. |
| Store any configurations such as processing rules or settings for a user or an organization's account. | Add processing request to each request/instruction. |
| Direct event handling of asset creation events from storage systems and processing completed notifications, and errors. | Use [!DNL Adobe I/O] Events and other methods. |

-->

>[!MORELIKETHIS]
>
>* [使用中的资源微服务进行资源处理概述 [!DNL Adobe Experience Manager] as a [!DNL Cloud Service]](https://experienceleague.adobe.com/zh-hans/docs/experience-manager-cloud-service/content/assets/asset-microservices-overview).
>* [Adobe Developer App Builder文档](https://developer.adobe.com/app-builder/docs/overview).
>* [支持的处理文件格式](https://experienceleague.adobe.com/en/docs/experience-manager-cloud-service/content/assets/file-format-support).

<!-- **TBD:**
* Clarify the service can only be used within AEM as Cloud Service. The docs provided as context for custom application developers. Not to be used as a standalone service.
  ** and API as that plays a role in custom applications (accepting standard params, invoking Nui itself in the future, etc. (this is an outlook))

* link to aem as cloud service docs on asset ingestion and customization with processing profiles.
-->
