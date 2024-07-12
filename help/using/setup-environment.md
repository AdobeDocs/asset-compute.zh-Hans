---
title: 设置 [!DNL Asset Compute Service]所需的开发环境
description: 为 [!DNL Asset Compute Service] 设置开发人员环境，以开始创建和测试自定义代码。
exl-id: 91c12889-01d8-4757-9bdd-f73c491cd9d5
source-git-commit: c6f747ebd6d1b17834f1af0837609a148804f8a9
workflow-type: tm+mt
source-wordcount: '324'
ht-degree: 0%

---

# 设置开发人员环境 {#create-dev-environment}

要创建允许您为[!DNL Asset Compute Service]开发的设置，请按照以下要求和说明操作。

1. [获取[!DNL Adobe Developer App Builder]的访问权限和凭据](https://developer.adobe.com/app-builder/docs/getting_started/#acquire-access-and-credentials)。

1. [设置本地环境](https://developer.adobe.com/app-builder/docs/getting_started/#local-environment-set-up)和所需的工具。

1. 还有其他一些工具可帮助您开始顺利开发，其中包括：

   * [Git](https://git-scm.com/)
   * [Docker桌面](https://www.docker.com/get-started)
   * [NodeJS](https://nodejs.org) （v14 LTS，不建议使用奇数版本）和[NPM](https://www.npmjs.com)。 OS X HomeBrew的用户可以`brew install node`同时安装这两者。 否则，请从[NodeJS下载页面](https://nodejs.org/en/)下载
   * IDE对NodeJS很有用，Adobe建议使用[Visual Studio Code (VS Code)](https://code.visualstudio.com)，因为它是调试器支持的IDE。 可以将任何其他IDE用作代码编辑器，但尚不支持高级使用（例如，调试器）
   * 安装最新的Adobe[[!DNL aio-cli]](https://github.com/adobe/aio-cli) (`aio`)
   <!-- - install using `npm install -g @adobe/aio-cli@7.1.0` -->

1. 确保满足[先决条件](/help/using/understand-extensibility.md#prerequisites-and-provisioning)

<!--
>[!NOTE]
>
>For now, use [!DNL Adobe I/O] CLI v7.1.0 of and do not use [!DNL Adobe I/O] CLI v8.
-->

## 设置App Builder项目 {#create-App-Builder-project}

1. 确保[!DNL Experience Cloud]组织中有系统管理员或开发人员角色。 系统管理员以[Admin Console](https://adminconsole.adobe.com/overview)设置此角色。

1. 登录到[Adobe Developer Console](https://developer.adobe.com/console/user/servicesandapis)。 确保您作为[!DNL Cloud Service]集成属于与[!DNL Experience Manager]相同的[!DNL Experience Cloud]组织。 有关Adobe Developer Console的详细信息，请转到[控制台文档](https://developer.adobe.com/developer-console/docs/guides/)。

1. [创建一个App Builder项目](https://developer.adobe.com/app-builder/docs/getting_started/first_app/)。 单击&#x200B;**[!UICONTROL 从模板创建新项目]** > **[!UICONTROL 项目]**。 选择App Builder。 它将创建一个新的App Builder项目，该项目具有两个工作区：`Production`和`Stage`。 根据需要添加其他工作区，例如`Development`。

1. 在App Builder项目中，选择工作区并订阅Asset compute所需的服务。 单击&#x200B;**添加到项目** > **API**&#x200B;并添加`Asset Compute`、`IO Events`和`IO Events Management`服务。 添加第一个API时，它会提示您创建私钥。 将此信息保存在计算机上，因为您需要此密钥才能使用开发人员工具测试自定义应用程序。

## 下一步 {#next-step}

设置好环境后，您便可以[创建自定义应用程序](develop-custom-application.md)。

<!-- More ideas:
 
* Any steps in the beginning that lead to gotchas later should be called out for caution? For example,
  * don't change some defaults initially
  * know risks when deviating from standard path
  * naming conventions to follow
  * Retrieve and format credentials (YAML file details)

TBD: When aio-cli v8 bugs are resolved, update the AIO CLI install command to remove v7.x reference and instruct users to use the latest version. See CQDOC-18346.

-->
