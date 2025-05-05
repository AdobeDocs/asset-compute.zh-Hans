---
title: 部署 [!DNL Asset Compute Service] 自定义应用程序
description: 部署 [!DNL Asset Compute Service] 自定义应用程序。
exl-id: a68d4f59-8a8f-43b2-8bc6-19320ac8c9ef
source-git-commit: f15b9819d3319d22deccdf7e39c0f72728baaa39
workflow-type: tm+mt
source-wordcount: '170'
ht-degree: 0%

---

# 部署自定义应用程序 {#deploy-custom-application}

要部署应用程序，请使用[aio应用程序部署](https://github.com/adobe/aio-cli#aio-appdeploy)命令。 在终端中，命令会显示一个用于访问自定义应用程序的URL。 URL的格式为`https://[namespace].adobeio-static.net/api/v1/web/[appname]-[appversion]/[workername]`。

若要在不重新部署应用程序的情况下获取相同的URL，请使用[`aio app get-url`](https://github.com/adobe/aio-cli#aio-app-get-url-action)命令。

在 [!DNL Experience Manager] as a [!DNL Cloud Service][&#128279;](https://experienceleague.adobe.com/zh-hans/docs/experience-manager-cloud-service/content/assets/manage/asset-microservices-configure-and-use)的处理配置文件中使用URL将您的应用程序与[!DNL Experience Manager]作为[!DNL Cloud Service]集成。

确保您的App Builder项目和工作区对应到要在其中使用操作的[!DNL Experience Manager] as a [!DNL Cloud Service]环境。 它具有不同的开发、暂存和生产环境。 您可以通过检查在Adobe Developer App Builder应用程序根目录的ENV文件中定义的`AIO_runtime_*`凭据来验证环境。 例如，要部署到`Stage`工作区，`AIO_runtime_namespace`的格式为`xxxxxx_xxxxxxxxx_stage`。 要将作为[!DNL Cloud Service]生产环境与[!DNL Experience Manager]集成，请使用Adobe Developer App Builder `Production`工作区中的应用程序URL。

>[!CAUTION]
>
>请勿在关键[!DNL Experience Manager]环境中使用个人工作区。

>[!MORELIKETHIS]
>
>* [了解并管理 [!DNL Experience Manager] as a [!DNL Cloud Service]](https://experienceleague.adobe.com/en/docs/experience-manager-cloud-service/content/implementing/using-cloud-manager/manage-environments)中的环境。
