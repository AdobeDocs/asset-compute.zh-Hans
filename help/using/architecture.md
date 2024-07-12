---
title: ' [!DNL Asset Compute Service]的架构'
description: ' [!DNL Asset Compute Service] API、应用程序和SDK如何协作以提供云原生资产处理服务。'
exl-id: 658ee4b7-5eb1-4109-b263-1b7d705e49d6
source-git-commit: f15b9819d3319d22deccdf7e39c0f72728baaa39
workflow-type: tm+mt
source-wordcount: '478'
ht-degree: 0%

---

# [!DNL Asset Compute Service]的架构 {#overview}

[!DNL Asset Compute Service]基于无服务器Adobe[!DNL `I/O Runtime`]平台构建。 它为Assets提供Adobe Sensei内容服务支持。 向调用客户端（仅支持[!DNL Experience Manager]作为[!DNL Cloud Service]）提供它为该资源搜索的Adobe Sensei生成的信息。 返回的信息采用JSON格式。

通过创建基于[!DNL Adobe Developer App Builder]的自定义应用程序，[!DNL Asset Compute Service]可扩展。 这些自定义应用程序是[!DNL Project Adobe Developer App Builder]个Headless应用程序，并执行添加自定义转换工具或调用外部API以执行图像操作等任务。

[!DNL Project Adobe Developer App Builder]是一个框架，用于在Adobe[!DNL `I/O Runtime`]上生成和部署自定义Web应用程序。 要创建自定义应用程序，开发人员可以利用[!DNL React Spectrum](Adobe的UI工具包)、创建微服务、创建自定义事件和编排API。 请参阅Adobe Developer App Builder](https://developer.adobe.com/app-builder/docs/overview)的[文档。

该体系结构所基于的基础包括：

* 仅包含给定任务所需内容的应用程序的模块化使得应用程序可以相互分离并保持轻量级。

* [!DNL Adobe I/O]运行时的无服务器概念带来了许多好处：异步、高度可扩展、隔离、基于作业的处理，非常适合资产处理。

* 二进制云存储使用预签名的URL引用，提供了单独存储和访问资产文件和演绎版所需的功能，无需对存储具有完全访问权限。 传输加速、CDN缓存和与云存储一起定位计算应用程序允许最佳的低延迟内容访问。 AWS和Azure云均受支持。

![Asset compute服务的架构](assets/architecture-diagram.png)

*图： [!DNL Asset Compute Service]的体系结构以及它如何与[!DNL Experience Manager]、存储和处理应用程序集成。*

该架构由以下部分组成：

* **API和协调层**&#x200B;接收请求（JSON格式），这些请求指示服务将源资源转换为多个演绎版。 这些请求是异步执行的，且会返回一个作为作业ID的激活ID。 指令是纯声明性的，对于所有标准处理工作（例如，缩略图生成、文本提取），使用者仅指定所需的结果，而不指定处理特定呈现的应用程序。 通用API功能（如身份验证、分析、速率限制）在服务前使用AdobeAPI网关处理，并管理所有到[!DNL Adobe I/O]运行时的请求。 应用程序路由由协调层动态完成。 客户端为特定呈现版本定义自定义应用程序，这些应用程序带有自己的一组唯一参数。 应用程序执行可以完全并行，因为它们是Adobe[!DNL `I/O Runtime`]中的单独的无服务器函数。

* **用于处理专门处理特定类型文件格式或目标呈现的资源的应用程序**。 从概念上讲，应用程序类似于UNIX®管道概念：输入文件被转换为一个或多个输出文件。

* **一个[公共应用程序库](https://github.com/adobe/asset-compute-sdk)**&#x200B;处理公共任务。 例如，下载源文件、上传演绎版、错误报告、事件发送和监控。 此设计确保应用程序开发保持简单明了，遵循无服务器概念，交互仅限于本地文件系统。

<!-- TBD:

* About the YAML file?
* minimize description to custom applications
* remove all internal stuff (e.g. Photoshop application, API Gateway) from text and diagram
* update diagram to focus on 3rd party custom applications ONLY
* Explain important transactions/handshakes?
* Flow of assets/control? See the illustration on the Nui diagrams wiki.
* Illustrations. See the SVG shared by Alex.
* Exceptions? Limitations? Call-outs? Gotchas?
* Do we want to add what basic processing is not available currently, that is expected by existing AEM customers?
-->
