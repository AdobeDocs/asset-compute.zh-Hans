---
title: 疑难解答 [!DNL Asset Compute Service]
description: 使用 [!DNL Asset Compute Service]对自定义应用程序进行故障排除和调试。
exl-id: 017fff91-e5e9-4a30-babf-5faa1ebefc2f
source-git-commit: c6f747ebd6d1b17834f1af0837609a148804f8a9
workflow-type: tm+mt
source-wordcount: '273'
ht-degree: 0%

---

# 疑难解答 {#troubleshoot}

以下是一般故障排除提示，可帮助您排除Asset compute服务故障：

* 确保JavaScript应用程序在启动时不会崩溃。 此类崩溃通常与缺少库或依赖关系有关。
* 请确保要安装的所有依赖项都在应用程序的`package.json`文件中引用。
* 确保任何因失败时清理而导致的错误不会生成隐藏原始问题的错误。

* 首次使用新的[!DNL Asset Compute Service]集成启动开发人员工具时，如果未完全设置Asset compute事件日志，则第一个处理请求可能会失败。 请等待一段时间以设置日志，然后再发送另一个请求。
* 确保您的Adobe[!DNL `I/O Project`]和Workspace中包含所有必需的APIAsset compute、Adobe[!DNL I/O Events]、事件管理和运行时，以避免`/register`或`/process`请求错误。

## 通过Adobe[!DNL aio-cli]登录问题 {#login-via-aio-cli}

如果您在通过Adobe [!DNL aio-cli][&#128279;](https://developer.adobe.com/app-builder/docs/getting_started/first_app/#3-signing-in-from-cli)登录[!DNL Adobe Developer Console] 时遇到问题，请手动添加开发、测试和部署自定义应用程序所需的凭据：

1. 导航到[Adobe Developer Console](https://developer.adobe.com/console/user/servicesandapis)上的Adobe Developer App Builder项目和工作区，然后从右上角按&#x200B;**[!UICONTROL 下载]**。 打开此文件并将此JSON保存到您计算机上的安全位置。

1. 导航到Adobe Developer App Builder应用程序中的环境文件。

1. 添加Adobe[!DNL I/O Runtime]凭据。 从下载的JSON获取Adobe[!DNL I/O Runtime]凭据。 凭据位于`project.workspace.services.runtime`下。 在`AIO_runtime_XXX`变量中添加[!DNL Adobe I/O]运行时凭据：

   ```json
   AIO_runtime_auth=
   AIO_runtime_namespace=
   ```

1. 在步骤1中添加已下载JSON的绝对路径：

   ```json
       ASSET_COMPUTE_INTEGRATION_FILE_PATH=
   ```

1. 设置开发人员工具所需的[必需凭据](develop-custom-application.md)的其余部分。

<!-- TBD for later:
Add any best practices for developers in this section:
* Any items to take care of when creating projects.
* Any naming conventions, reserved keywords, etc.?
* Any terms that can become a source of confusion later based on our OOTB naming.

* If required, add limitations for custom applications and spin those off as best practices.
* Do NOT borrow any content from https://git.corp.adobe.com/nui/nui/blob/master/doc/worker_api.md. It is outdated and irrelevant for 3rd party custom applications.
-->
