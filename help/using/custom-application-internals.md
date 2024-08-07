---
title: 了解自定义应用程序的工作情况
description: ' [!DNL Asset Compute Service] 自定义应用程序的内部工作，以帮助了解其工作方式。'
exl-id: a3ee6549-9411-4839-9eff-62947d8f0e42
source-git-commit: f15b9819d3319d22deccdf7e39c0f72728baaa39
workflow-type: tm+mt
source-wordcount: '691'
ht-degree: 0%

---

# 自定义应用程序的内部结构 {#how-custom-application-works}

使用下图了解在客户端使用自定义应用程序处理数字资产时的端到端工作流程。

![自定义应用程序工作流](assets/customworker.svg)

*图：使用Adobe[!DNL Asset Compute Service].*&#x200B;处理资源时涉及的步骤

## 注册 {#registration}

客户端必须在第一次向[`/process`](api.md#process-request)发出请求之前调用[`/register`](api.md#register)一次，以便设置和检索日志URL以接收AdobeAsset compute的Adobe[!DNL I/O Events]事件。

```sh
curl -X POST \
  https://asset-compute.adobe.io/register \
  -H "x-ims-org-id: $ORG_ID" \
  -H "x-gw-ims-org-id: $ORG_ID" \
  -H "Authorization: Bearer $JWT_TOKEN" \
  -H "x-api-key: $API_KEY"
```

[`@adobe/asset-compute-client`](https://github.com/adobe/asset-compute-client#usage) JavaScript库可以在NodeJS应用程序中使用，以处理从注册、处理到异步事件处理的所有必要步骤。 有关所需标头的详细信息，请参阅[身份验证和授权](api.md)。

## 正在处理 {#processing}

客户端发送了[处理](api.md#process-request)请求。

```sh
curl -X POST \
  https://asset-compute.adobe.io/process \
  -H "x-ims-org-id: $ORG_ID" \
  -H "x-gw-ims-org-id: $ORG_ID" \
  -H "Authorization: Bearer $JWT_TOKEN" \
  -H "x-api-key: $API_KEY" \
  -d "<RENDITION_JSON>
```

客户端负责使用预签名URL正确设置演绎版格式。 可在NodeJS应用程序中使用[`@adobe/node-cloud-blobstore-wrapper`](https://github.com/adobe/node-cloud-blobstore-wrapper#presigned-urls) JavaScript库对URL进行预签名。 目前，该库仅支持Azure Blob Storage和AWS S3容器。

处理请求返回可用于轮询[!DNL Adobe I/O]事件的`requestId`。

下面是一个自定义应用程序处理请求示例。

```json
{
    "source": "https://www.adobe.com/some-source-file.jpg",
    "renditions" : [
        {
            "worker": "https://my-project-namespace.adobeioruntime.net/api/v1/web/my-namespace-version/my-worker",
            "name": "rendition1.jpg",
            "target": "https://some-presigned-put-url-for-rendition1.jpg",
        }
    ],
    "userData": {
        "my-asset-id": "1234567890"
    }
}
```

[!DNL Asset Compute Service]将自定义应用程序演绎版请求发送到自定义应用程序。 它使用HTTPPOST访问提供的应用程序URL，该URL是来自App Builder的安全Web操作URL。 所有请求都使用HTTPS协议来最大限度地提高数据安全性。

自定义应用程序使用的[Asset computeSDK](https://github.com/adobe/asset-compute-sdk#adobe-asset-compute-worker-sdk)处理HTTPPOST请求。 它还处理源下载、上传演绎版、发送Adobe[!DNL I/O Events]和错误处理。

<!-- TBD: Add the application diagram. -->

### 应用程序代码 {#application-code}

自定义代码只需提供回调即可获取本地可用的源文件(`source.path`)。 `rendition.path`是放置资产处理请求的最终结果的位置。 自定义应用程序使用回调将本地可用的源文件转换为使用传入的名称(`rendition.path`)的格式副本文件。 自定义应用程序必须写入`rendition.path`才能创建演绎版：

```javascript
const { worker } = require('@adobe/asset-compute-sdk');
const fs = require('fs').promises;

// worker() is the entry point in the SDK "framework".
// The asynchronous function defined is the rendition callback.
exports.main = worker(async (source, rendition) => {

    // Tip: custom worker parameters are available in rendition.instructions.
    console.log(rendition.instructions.name); // should print out `rendition.jpg`.

    // Simplest example: copy the source file to the rendition file destination so as to transfer the asset as is without processing.
    await fs.copyFile(source.path, rendition.path);
});
```

### 下载源文件 {#download-source}

自定义应用程序仅处理本地文件。 [Asset computeSDK](https://github.com/adobe/asset-compute-sdk#adobe-asset-compute-worker-sdk)处理源文件的下载。

### 创建演绎版 {#rendition-creation}

SDK为每个节目调用异步[节目回调函数](https://github.com/adobe/asset-compute-sdk#rendition-callback-for-worker-required)。

回调函数具有对[源](https://github.com/adobe/asset-compute-sdk#source)和[解释](https://github.com/adobe/asset-compute-sdk#rendition)对象的访问权限。 `source.path`已存在，是源文件的本地副本的路径。 `rendition.path`是必须存储已处理演绎版的路径。 除非设置了[disableSourceDownload标志](https://github.com/adobe/asset-compute-sdk#worker-options-optional)，否则应用程序必须完全使用`rendition.path`。 否则，SDK将无法找到或识别演绎版文件，并将失败。

通过对示例的过度简化，说明了自定义应用程序的剖析。 应用程序只会将源文件复制到节目目标。

有关演绎版回调参数的更多信息，请参阅[Asset computeSDK API](https://github.com/adobe/asset-compute-sdk#api-details)。

### 上传节目 {#upload-rendition}

在每个演绎版创建并存储在`rendition.path`提供的路径的文件中后，[Asset computeSDK](https://github.com/adobe/asset-compute-sdk#adobe-asset-compute-worker-sdk)会将每个演绎版上传到云存储(AWS或Azure)。 当且仅当传入请求具有多个指向同一应用程序URL的演绎版时，自定义应用程序才会同时获取多个演绎版。 上传到云存储是在每个演绎版之后并为下一个演绎版运行回调之前完成的。

`batchWorker()`具有不同的行为。 它会处理所有演绎版，并且只有在处理完所有演绎版后，才会上传它们。

## [!DNL Adobe I/O]个事件 {#aio-events}

SDK为每个呈现版本发送Adobe[!DNL I/O Events]。 这些事件的类型是`rendition_created`或`rendition_failed`，具体取决于结果。 有关详细信息，请参阅[Asset compute异步事件](api.md#asynchronous-events)。

## 接收[!DNL Adobe I/O]个事件 {#receive-aio-events}

客户端根据其使用逻辑轮询Adobe[!DNL I/O Events]日志。 初始日记账URL是`/register` API响应中提供的URL。 可以使用事件中存在的`requestId`识别事件，该值与在`/process`中返回的相同。 每个演绎版都有一个单独的事件，在上传（或失败）演绎版后立即发送该事件。 当客户端收到匹配事件时，可以显示或以其他方式处理生成的演绎版。

JavaScript库[`asset-compute-client`](https://github.com/adobe/asset-compute-client#usage)使用`waitActivation()`方法简化日志轮询以获取所有事件。

```javascript
const events = await assetCompute.waitActivation(requestId);
await Promise.all(events.map(event => {
    if (event.type === "rendition_created") {
        // get rendition from cloud storage location
    }
    else if (event.type === "rendition_failed") {
        // failed to process
    }
    else {
        // other event types
        // (could be added in the future)
    }
}));
```

有关如何获取日志事件的详细信息，请参阅Adobe[[!DNL I/O Events] API](https://developer.adobe.com/events/docs/guides/api/journaling_api/)。

<!-- TBD:
* Illustration of the controls/data flow.
* Basic overview, in text and not code, of how an application works.
-->
