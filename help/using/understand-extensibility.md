---
title: 了解如何扩展 [!DNL Asset Compute Service]
description: 何时以及如何扩展 [!DNL Asset Compute Service] 功能以进行自定义资产处理。
exl-id: 3b903364-34cc-44d5-9a03-24a0102cf85d
source-git-commit: c6f747ebd6d1b17834f1af0837609a148804f8a9
workflow-type: tm+mt
source-wordcount: '229'
ht-degree: 0%

---

# 可扩展性简介 {#introduction-to-extensibilty}

[在 [!DNL Experience Manager] as a [!DNL Cloud Service]](https://experienceleague.adobe.com/zh-hans/docs/experience-manager-cloud-service/content/assets/asset-microservices-overview)中处理用户档案，可满足许多演绎版要求，如转换为格式和调整图像大小。 更复杂的业务需求可能需要一种定制的解决方案，以满足组织的需求。 通过创建从[!DNL Experience Manager]中的处理配置文件调用的自定义应用程序，可以扩展[!DNL Asset Compute Service]。 这些自定义应用程序符合[支持的用例](https://experienceleague.adobe.com/zh-hans/docs/experience-manager-cloud-service/content/assets/manage/asset-microservices-configure-and-use)。

>[!NOTE]
>
>[!DNL Asset Compute Service]仅可用于[!DNL Experience Manager]作为[!DNL Cloud Service]。

自定义应用程序是Headless [Adobe Developer App Builder](https://github.com/AdobeDocs/app-builder)应用程序。 通过[Asset computeSDK](https://github.com/adobe/asset-compute-sdk)和Adobe Developer App Builder开发人员工具，可以轻松扩展包含自定义应用程序的[!DNL Asset Compute Service]。 这些工具让开发人员专注于业务逻辑。 创建自定义应用程序与创建普通无服务器Adobe[!DNL I/O Runtime]操作一样简单。 它是单个Node.js JavaScript函数。 [基本自定义应用程序示例](https://github.com/adobe/asset-compute-example-workers/blob/master/projects/worker-basic/worker-basic.js)对此进行了说明。

## 先决条件和配置要求 {#prerequisites-and-provisioning}

确保满足以下先决条件：

* Adobe Developer App Builder工具安装在您的计算机上。
* [!DNL Experience Cloud]组织。 有关详细信息，请转到[启动App Builder历程](https://developer.adobe.com/app-builder/docs/getting_started/#acquire-access-and-credentials)。
* Experience Organization必须将[!DNL Experience Manager]作为[!DNL Cloud Service]启用。
* [!DNL Adobe Experience Cloud]组织是[!DNL Adobe Developer App Builder]开发人员偷看计划的一部分。 转到[如何申请访问权限](https://developer.adobe.com/app-builder/docs/overview/getting_access)。
* 确保开发人员在组织中拥有开发人员角色或管理员权限。
* 确保已在本地安装Adobe[[!DNL aio-cli]](https://github.com/adobe/aio-cli)。

<!-- TBD for later:

* What all accesses and licenses are required?
* What all permissions are required to create, debug, and deploy custom applications?
* How do developers get access and provision the required apps?
* What is repository management?
* Anything on security and data transfer?
* What about handling personal or sensitive information?
* Custom application SLA is dependent on SLAs of various services it depends on.
* Document how the devs can get to know the KPIs of their custom applications. The KPIs are dependent on the performance at Adobe's side, amongst other things.
-->
