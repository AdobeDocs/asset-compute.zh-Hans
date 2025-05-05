---
title: "[!DNL Asset Compute Service] HTTP API"
description: "[!DNL Asset Compute Service] HTTP API用于创建自定义应用程序。"
exl-id: 4b63fdf9-9c0d-4af7-839d-a95e07509750
source-git-commit: f15b9819d3319d22deccdf7e39c0f72728baaa39
workflow-type: tm+mt
source-wordcount: '2862'
ht-degree: 2%

---

# [!DNL Asset Compute Service] HTTP API {#asset-compute-http-api}

API的使用仅限于开发目的。 API在开发自定义应用程序时作为上下文提供。 [!DNL Adobe Experience Manager]作为[!DNL Cloud Service]使用该API将处理信息传递给自定义应用程序。 有关详细信息，请参阅[使用资产微服务和处理配置文件](https://experienceleague.adobe.com/zh-hans/docs/experience-manager-cloud-service/content/assets/manage/asset-microservices-configure-and-use)。

>[!NOTE]
>
>[!DNL Asset Compute Service]仅可用于[!DNL Experience Manager]作为[!DNL Cloud Service]。

[!DNL Asset Compute Service] HTTP API的任何客户端都必须遵循此高级流程：

1. 客户端被设置为IMS组织中的[!DNL Adobe Developer Console]项目。 每个单独的客户端（系统或环境）都需要其自己的单独项目来分离事件数据流。

1. 客户端使用[JWT（服务帐户）身份验证](https://developer.adobe.com/developer-console/docs/guides/)为技术帐户生成访问令牌。

1. 客户端只调用[`/register`](#register)一次以检索日志URL。

1. 客户端为要为其生成演绎版的每个资产调用[`/process`](#process-request)。 调用是异步调用。

1. 客户端定期轮询日志以[接收事件](#asynchronous-events)。 成功处理节目（`rendition_created`事件类型）或发生错误（`rendition_failed`事件类型）时，它会接收每个请求节目的事件。

通过[adobe-asset-compute-client](https://github.com/adobe/asset-compute-client)模块，可以轻松使用Node.js代码中的API。

## 身份验证和授权 {#authentication-and-authorization}

所有API都需要访问令牌身份验证。 请求必须设置以下标头：

1. 通过[JWT交换](https://developer.adobe.com/developer-console/docs/guides/)从Adobe Developer Console项目收到的带持有者令牌的`Authorization`标头，它是技术帐户令牌。 [作用域](#scopes)记录如下。

<!-- TBD: Change the existing URL to a new path when a new path for docs is available. The current path contains master word that is not an inclusive term. Logged ticket in Adobe I/O's GitHub repo to get a new URL.
-->

1. 具有IMS组织ID的`x-gw-ims-org-id`标头。

1. [!DNL Adobe Developers Console]项目中具有客户端ID的`x-api-key`。

### 范围 {#scopes}

确保访问令牌的以下范围：

* `openid`
* `AdobeID`
* `asset_compute`
* `read_organizations`
* `event_receiver`
* `event_receiver_api`
* `adobeio_api`
* `additional_info.roles`
* `additional_info.projectedProductContext`

这些范围要求将[!DNL Adobe Developer Console]项目订阅到`Asset Compute`、`I/O Events`和`I/O Management API`服务。 各个范围的划分如下：

* 基本
   * 范围： `openid,AdobeID`

* asset compute
   * metascope： `asset_compute_meta`
   * 范围： `asset_compute,read_organizations`

* Adobe [!DNL `I/O Events`]
   * metascope： `event_receiver_api`
   * 范围： `event_receiver,event_receiver_api`

* Adobe [!DNL `I/O Management API`]
   * metascope： `ent_adobeio_sdk`
   * 范围： `adobeio_api,additional_info.roles,additional_info.projectedProductContext`

## 注册 {#register}

[!DNL Asset Compute service]的每个客户端 — 订阅了服务的唯一[!DNL Adobe Developer Console]项目 — 必须[注册](#register-request)，然后才能发出处理请求。 注册步骤会返回从演绎版处理中检索异步事件所需的唯一事件日志。

在其生命周期结束时，客户端可以[取消注册](#unregister-request)。

### 注册请求 {#register-request}

此API调用设置[!DNL Asset Compute]客户端并提供事件日志URL。 此过程是一个幂等操作，每个客户端只需调用一次。 可以再次调用它以检索日志URL。

| 参数 | 价值 |
|--------------------------|------------------------------------------------------|
| 方法 | `POST` |
| 路径 | `/register` |
| 标头`Authorization` | 所有[授权相关标头](#authentication-and-authorization)。 |
| 标头`x-request-id` | 可选，由客户端设置，用于跨系统处理请求的唯一端到端标识符。 |
| 请求正文 | 必须为空。 |

### 注册响应 {#register-response}

| 参数 | 价值 |
|-----------------------|------------------------------------------------------|
| MIME类型 | `application/json` |
| 标头`X-Request-Id` | 与`X-Request-Id`请求标头相同或生成一个唯一标头。 用于标识跨系统的请求或支持请求，或同时使用两者。 |
| 响应正文 | 包含`journal`、`ok`或`requestId`字段的JSON对象。 |

HTTP状态代码为：

* **200成功**：请求成功时。 `journal` URL接收有关通过`/process`启动的异步处理结果的通知。 它在成功完成时发出`rendition_created`个事件的警报，如果进程失败，则发出`rendition_failed`个事件的警报。

  ```json
  {
      "ok": true,
      "journal": "https://api.adobe.io/events/organizations/xxxxx/integrations/xxxx/xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx",
      "requestId": "1234567890"
  }
  ```

* **401 Unauthorized**：当请求没有有效的[身份验证](#authentication-and-authorization)时发生。 例如，访问令牌无效或API密钥无效。

* **403 Forbidden**：当请求没有有效的[授权](#authentication-and-authorization)时发生。 示例可能是有效的访问令牌，但Adobe Developer Console项目（技术帐户）未订阅所有必需服务。

* **429请求太多**：发生在此客户端或系统过载时。 客户端应使用[指数回退](https://en.wikipedia.org/wiki/Exponential_backoff)重试。 正文为空。
* **4xx错误**：出现任何其他客户端错误时注册失败。 通常，会返回此类的JSON响应，但不一定会发生所有错误：

  ```json
  {
      "ok": false,
      "requestId": "1234567890",
      "message": "error message"
  }
  ```

* **5xx错误**：发生任何其他服务器端错误且注册失败时。 通常，会返回此类的JSON响应，但不一定会发生所有错误：

  ```json
  {
      "ok": false,
      "requestId": "1234567890",
      "message": "error message"
  }
  ```

### 取消注册请求 {#unregister-request}

此API调用取消注册[!DNL Asset Compute]客户端。 取消注册后，无法再调用`/process`。 对未注册的客户端或尚未注册的客户端使用API调用会返回`404`错误。

| 参数 | 价值 |
|--------------------------|------------------------------------------------------|
| 方法 | `POST` |
| 路径 | `/unregister` |
| 标头`Authorization` | 所有[授权相关标头](#authentication-and-authorization)。 |
| 标头`x-request-id` | 可选。 客户端可以将其设置为跨系统的处理请求的唯一端到端标识符。 |
| 请求正文 | 空。 |

### 注销响应 {#unregister-response}

| 参数 | 价值 |
|-----------------------|------------------------------------------------------|
| MIME类型 | `application/json` |
| 标头`X-Request-Id` | 与`X-Request-Id`请求标头相同或生成一个唯一标头。 用于标识跨系统的请求或支持请求。 |
| 响应正文 | 包含`ok`和`requestId`字段的JSON对象。 |

状态代码为：

* **200成功**：找到并删除注册和日记时发生。

  ```json
  {
      "ok": true,
      "requestId": "1234567890"
  }
  ```

* **401 Unauthorized**：当请求没有有效的[身份验证](#authentication-and-authorization)时发生。 例如，访问令牌无效或API密钥无效。

* **403 Forbidden**：当请求没有有效的[授权](#authentication-and-authorization)时发生。 示例可能是有效的访问令牌，但Adobe Developer Console项目（技术帐户）未订阅所有必需服务。

* **404未找到**：当提供的凭据未注册或无效时，将出现此状态。

  ```json
  {
      "ok": true,
      "requestId": "1234567890"
  }
  ```

* **429请求太多**：在系统过载时发生。 客户端应使用[指数回退](https://en.wikipedia.org/wiki/Exponential_backoff)重试。 正文为空。

* **4xx错误**：发生任何其他客户端错误时注销失败。 通常，会返回此类的JSON响应，但不一定会发生所有错误：

  ```json
  {
      "ok": false,
      "requestId": "1234567890",
      "message": "error message"
  }
  ```

* **5xx错误**：发生任何其他服务器端错误且注册失败时。 通常，会返回此类的JSON响应，但不一定会发生所有错误：

  ```json
  {
      "ok": false,
      "requestId": "1234567890",
      "message": "error message"
  }
  ```

## 流程请求 {#process-request}

`process`操作提交的作业将根据请求中的说明将源资源转换为多个演绎版。 有关成功完成（事件类型`rendition_created`）或任何错误（事件类型`rendition_failed`）的通知将发送到事件日志，该日志必须在发出任意数量的`/process`请求之前使用[`/register`](#register)检索一次。 格式不正确的请求立即失败，并显示400错误代码。

使用URL引用二进制文件，例如Amazon AWS S3预签名URL或Azure Blob Storage SAS URL。 用于读取`source`资源(`GET` URL)和写入演绎版(`PUT` URL)。 客户端负责生成这些预签名URL。

| 参数 | 价值 |
|--------------------------|------------------------------------------------------|
| 方法 | `POST` |
| 路径 | `/process` |
| MIME类型 | `application/json` |
| 标头`Authorization` | 所有[授权相关标头](#authentication-and-authorization)。 |
| 标头`x-request-id` | 可选。 客户端可以设置唯一的端到端标识符来跟踪跨系统的处理请求。 |
| 请求正文 | 必须采用如下所述的流程请求JSON格式。 它提供了有关要处理的资产以及要生成的演绎版的说明。 |

### 处理请求JSON {#process-request-json}

`/process`的请求正文是一个具有此高级架构的JSON对象：

```json
{
    "source": "",
    "renditions" : []
}
```

可用字段包括：

| 名称 | 类型 | 描述 | 示例 |
|--------------|----------|-------------|---------|
| `source` | `string` | 已处理的源资源的URL。 可选，基于请求的演绎版格式（例如，`fmt=zip`）。 | `"http://example.com/image.jpg"` |
| `source` | `object` | 描述已处理的源资源。 请参阅下面的[Source对象字段](#source-object-fields)的说明。 基于请求的演绎版格式（例如，`fmt=zip`）的可选。 | `{"url": "http://example.com/image.jpg", "mimeType": "image/jpeg" }` |
| `renditions` | `array` | 要从源文件生成的演绎版。 每个节目对象都支持[节目指令](#rendition-instructions)。 必需。 | `[{ "target": "https://....", "fmt": "png" }]` |

`source`可以是视为URL的`<string>`，也可以是带有附加字段的`<object>`。 以下变量类似：

```json
"source": "http://example.com/image.jpg"
```

```json
"source": {
    "url": "http://example.com/image.jpg"
}
```

### Source对象字段 {#source-object-fields}

| 名称 | 类型 | 描述 | 示例 |
|-----------|----------|-------------|---------|
| `url` | `string` | 要处理的源资产的URL。 必需。 | `"http://example.com/image.jpg"` |
| `name` | `string` | Source资源文件名。 如果未检测到MIME类型，则可以使用名称为的文件扩展名。 其优先级高于URL路径中指定的文件名。 并且，其优先级高于二进制资源`content-disposition`标头中的文件名。 默认为“file”。 | `"image.jpg"` |
| `size` | `number` | Source资源文件大小（字节）。 优先于二进制资源的`content-length`标头。 | `10234` |
| `mimetype` | `string` | Source资源文件MIME类型。 优先于二进制资源的`content-type`标头。 | `"image/jpeg"` |

### 完整的`process`请求示例 {#complete-process-request-example}

```json
{
    "source": "https://www.adobe.com/content/dam/acom/en/lobby/lobby-bg-bts2017-logged-out-1440x860.jpg",
    "renditions" : [{
            "name": "image.48x48.png",
            "target": "https://some-presigned-put-url-for-image.48x48.png",
            "fmt": "png",
            "width": 48,
            "height": 48
        },{
            "name": "image.200x200.jpg",
            "target": "https://some-presigned-put-url-for-image.200x200.jpg",
            "fmt": "jpg",
            "width": 200,
            "height": 200
        },{
            "name": "cqdam.xmp.xml",
            "target": "https://some-presigned-put-url-for-cqdam.xmp.xml",
            "fmt": "xmp"
        },{
            "name": "cqdam.text.txt",
            "target": "https://some-presigned-put-url-for-cqdam.text.txt",
            "fmt": "text"
    }]
}
```

## 进程响应 {#process-response}

基于基本请求验证，`/process`请求立即返回成功或失败。 实际资产处理是异步进行的。

| 参数 | 价值 |
|-----------------------|------------------------------------------------------|
| MIME类型 | `application/json` |
| 标头`X-Request-Id` | 与`X-Request-Id`请求标头相同或生成一个唯一标头。 用于标识跨系统的请求或支持请求。 |
| 响应正文 | 包含`ok`和`requestId`字段的JSON对象。 |

状态代码：

* **200成功**：请求提交成功。 响应JSON包含`"ok": true`：

  ```json
  {
      "ok": true,
      "requestId": "1234567890"
  }
  ```

* **400无效请求**：如果请求结构不正确，例如，JSON有效负载中缺少必填字段。 响应JSON包含`"ok": false`：

  ```json
  {
      "ok": false,
      "requestId": "1234567890",
      "message": "error message"
  }
  ```

* **401未授权**：请求没有有效的[身份验证](#authentication-and-authorization)。 例如，访问令牌无效或API密钥无效。
* **403 Forbidden**：请求没有有效的[授权](#authentication-and-authorization)。 示例可能是有效的访问令牌，但Adobe Developer Console项目（技术帐户）未订阅所有必需服务。
* **429请求太多**：系统不堪重负时发生，可能是由于此特定客户端或整体需求所致。 客户端可以使用[指数回退](https://en.wikipedia.org/wiki/Exponential_backoff)重试。 正文为空。
* **4xx错误**：出现任何其他客户端错误时。 通常，会返回此类的JSON响应，但不一定会发生所有错误：

  ```json
  {
      "ok": false,
      "requestId": "1234567890",
      "message": "error message"
  }
  ```

* **5xx错误**：发生任何其他服务器端错误时。 通常，会返回此类的JSON响应，但不一定会发生所有错误：

  ```json
  {
      "ok": false,
      "requestId": "1234567890",
      "message": "error message"
  }
  ```

除&#x200B;*配置问题（如401或403）或无效请求（如400）外，大多数客户端可能倾向于在出现任何错误*&#x200B;时通过[指数回退](https://en.wikipedia.org/wiki/Exponential_backoff)重试同一请求。 除了429响应的常规速率限制外，临时服务中断或限制可能会导致5xx错误。 然后，建议您在一段时间后重试。

所有JSON响应（如果存在）都包含`requestId`，该值与`X-Request-Id`标头相同。 Adobe建议从标头中读取，因为它始终存在。 与处理请求相关的所有事件中也返回`requestId`作为`requestId`。 客户端不得对此字符串的格式进行任何假设。 它是一个不透明的字符串标识符。

## 选择加入后处理 {#opt-in-to-post-processing}

[Asset computeSDK](https://github.com/adobe/asset-compute-sdk)支持一组基本的图像后处理选项。 自定义工作进程可以通过将节目对象上的字段`postProcess`设置为`true`来显式选择加入后处理。

支持的用例包括：

* 裁切是矩形的演绎版，其限制由crop.w、crop.h、crop.x和crop.y定义。 裁剪详细信息在节目对象的`instructions.crop`字段中指定。
* 使用宽度、高度或两者来调整图像大小。 `instructions.width`和`instructions.height`在演绎版对象中定义它。 要仅使用宽度或高度调整大小，请仅设置一个值。 计算服务可节省宽高比。
* 设置JPEG图像的质量。 `instructions.quality`在演绎版对象中定义它。 质量级别为100表示最高质量，而数字越低表示质量下降。
* 创建交错图像。 `instructions.interlace`在演绎版对象中定义它。
* 设置DPI可通过调整应用于像素的缩放比例来调整渲染大小，以用于桌面出版目的。 `instructions.dpi`在演绎版对象中定义它以更改DPI分辨率。 但是，要调整图像大小以便以不同的分辨率显示相同的大小，请使用`convertToDpi`说明。
* 调整图像大小，使其渲染的宽度或高度在指定的目标分辨率(DPI)下与原始图像保持相同。 `instructions.convertToDpi`在演绎版对象中定义它。

## 为资源添加水印 {#add-watermark}

[Asset computeSDK](https://github.com/adobe/asset-compute-sdk)支持向PNG、JPEG、TIFF和GIF图像文件添加水印。 水印是按照演绎版上`watermark`对象中的演绎版说明添加的。

水印在演绎版后处理期间完成。 若要为资产添加水印，自定义工作进程[通过将演绎版对象上的字段`postProcess`设置为`true`来选择后处理](#opt-in-to-post-processing)。 如果辅助进程未选择加入，则不会应用水印，即使对请求中的演绎版对象设置了水印对象也是如此。

## 节目说明 {#rendition-instructions}

以下是[`/process`](#process-request)中`renditions`数组的可用选项。

### 常用字段 {#common-fields}

| 名称 | 类型 | 描述 | 示例 |
|-------------------|----------|-------------|---------|
| `fmt` | `string` | 对于文本提取，演绎版目标格式也可以为`text`；对于将XMP元数据提取为xml，目标格式可以为`xmp`。 查看[支持的格式](https://experienceleague.adobe.com/zh-hans/docs/experience-manager-cloud-service/content/assets/file-format-support) | `png` |
| `worker` | `string` | [自定义应用程序](develop-custom-application.md)的URL。 必须为`https://` URL。 如果存在此字段，则自定义应用程序会创建演绎版。 然后，在自定义应用程序中使用任何其他设置演绎版字段。 | `"https://1234.adobeioruntime.net`<br>`/api/v1/web`<br>`/example-custom-worker-master/worker"` |
| `target` | `string` | 应使用HTTPPUT将生成的演绎版上载到的URL。 | `http://w.com/img.jpg` |
| `target` | `object` | 生成的演绎版的多部分预签名URL上传信息。 此信息适用于具有此[多部分上传行为](https://jackrabbit.apache.org/oak/docs/apidocs/org/apache/jackrabbit/api/binary/BinaryUpload.html)的[AEM / Oak直接二进制上传](https://jackrabbit.apache.org/oak/docs/features/direct-binary-access.html)。<br>字段：<ul><li>`urls`：字符串数组，每个预签名部分URL一个</li><li>`minPartSize`：用于一个部分的最小大小= url</li><li>`maxPartSize`：用于一个部分的最大大小= url</li></ul> | `{ "urls": [ "https://part1...", "https://part2..." ], "minPartSize": 10000, "maxPartSize": 100000 }` |
| `userData` | `object` | 可选。 客户端控制保留的空间，并将其按原样传递到演绎版事件。 允许客户端添加自定义信息以标识演绎版事件。 在自定义应用程序中不得修改或依赖它，因为客户端可以随时随意更改它。 | `{ ... }` |

### 节目特定字段 {#rendition-specific-fields}

有关当前支持的文件格式的列表，请参阅[支持的文件格式](https://experienceleague.adobe.com/zh-hans/docs/experience-manager-cloud-service/content/assets/file-format-support)。

| 名称 | 类型 | 描述 | 示例 |
|-------------------|----------|-------------|---------|
| `*` | `*` | 可以添加[自定义应用程序](develop-custom-application.md)了解的高级自定义字段。 | |
| `embedBinaryLimit` | `number`字节 | 当演绎版的文件大小小于指定的值时，它包含在创建完成后发送的事件中。 允许嵌入的最大大小为32 KB （32 x 1024字节）。 如果演绎版的大小大于`embedBinaryLimit`限制，则将其放置在云存储中的某个位置，并且不会嵌入事件中。 | `3276` |
| `width` | `number` | 宽度（以像素为单位）。 仅适用于图像演绎版。 | `200` |
| `height` | `number` | 高度（像素）。 仅适用于图像演绎版。 | `200` |
|                   |          | 在以下情况下，将始终保持宽高比： <ul> <li> 同时指定了`width`和`height`，则在保持宽高比的同时，图像大小适合 </li><li> 如果仅指定`width`或`height`，则生成的图像将使用相应的维度，同时保持宽高比</li><li> 如果未指定`width`或`height`，则使用原始图像像素大小。 这取决于源类型。 对于某些格式(如PDF文件)，使用缺省大小。 可以有最大大小限制。</li></ul> | |
| `quality` | `number` | 指定`1`到`100`范围内的jpeg品质。 仅适用于图像演绎版。 | `90` |
| `xmp` | `string` | 仅由XMP元数据写回使用，它是base64编码的XMP，用于写回指定的演绎版。 | |
| `interlace` | `bool` | 通过将隔行PNG、GIF或渐进式JPEG设置为`true`来创建它。 它对其他文件格式没有影响。 | |
| `jpegSize` | `number` | JPEG文件的大致大小（字节）。 它覆盖任何`quality`设置。 它对其他格式没有影响。 | |
| `dpi` | `number` 或 `object` | 设置x和y DPI。 为简单起见，也可以将其设置为单个数字，该数字用于x和y。它对图像本身没有影响。 | `96` 或 `{ xdpi: 96, ydpi: 96 }` |
| `convertToDpi` | `number` 或 `object` | x和y DPI重新取样值，同时保持物理大小。 为简单起见，也可以将其设置为单个数字，该数字用于x和y。 | `96` 或 `{ xdpi: 96, ydpi: 96 }` |
| `files` | `array` | 要包含在ZIP存档中的文件列表(`fmt=zip`)。 每个条目可以是URL字符串或包含字段的对象：<ul><li>`url`：下载文件的URL</li><li>`path`：将文件存储在ZIP文件路径下</li></ul> | `[{ "url": "https://host/asset.jpg", "path": "folder/location/asset.jpg" }]` |
| `duplicate` | `string` | ZIP存档(`fmt=zip`)的重复处理。 默认情况下，存储在ZIP中同一路径下的多个文件会生成错误。 将`duplicate`设置为`ignore`只会导致存储第一个资产，忽略其余资产。 | `ignore` |
| `watermark` | `object` | 包含有关[水印](#watermark-specific-fields)的说明。 |  |

### 水印特定的字段 {#watermark-specific-fields}

PNG格式用作水印。

| 名称 | 类型 | 描述 | 示例 |
|-------------------|----------|-------------|---------|
| `scale` | `number` | 水印的比例，介于`0.0`和`1.0`之间。 `1.0`表示水印具有其原始比例(1:1)，较低的值会减小水印大小。 | 值`0.5`表示原始大小的一半。 |
| `image` | `url` | 用于添加水印的PNG文件的URL。 | |

## 异步事件 {#asynchronous-events}

当节目处理完成或发生错误时，会向Adobe[!DNL `I/O Events Journal`]发送一个事件。 客户端必须侦听通过[`/register`](#register)提供的日志URL。 日志响应包含一个`event`数组，该数组对于每个事件都包含一个对象，其中`event`字段包含实际事件有效负载。

[!DNL Asset Compute Service]的所有事件的Adobe[!DNL `I/O Events`]类型为`asset_compute`。 日志仅自动订阅此事件类型，无需根据[!DNL Adobe Developer]事件类型进行筛选。 服务特定的事件类型在事件的`type`属性中可用。

### 事件类型 {#event-types}

| 事件 | 描述 |
|---------------------|-------------|
| `rendition_created` | 每个成功处理和上传的节目均发送。 |
| `rendition_failed` | 为处理或上传失败的每个演绎版发送。 |

### 事件属性 {#event-attributes}

| 属性 | 类型 | 事件 | 描述 |
|-------------|----------|---------------|-------------|
| `date` | `string` | `*` | 按JavaScript [Date.toISOString()](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Date/toISOString)定义的简化扩展[ISO-8601](https://en.wikipedia.org/wiki/ISO_8601)格式发送事件的时间戳。 |
| `requestId` | `string` | `*` | 发送给`/process`的原始请求的请求ID，与`X-Request-Id`标头相同。 |
| `source` | `object` | `*` | `/process`请求的`source`。 |
| `userData` | `object` | `*` | 来自`/process`请求的`userData`演绎版（如果已设置）。 |
| `rendition` | `object` | `rendition_*` | 在`/process`中传递的对应节目对象。 |
| `metadata` | `object` | `rendition_created` | 节目的[元数据](#metadata)属性。 |
| `errorReason` | `string` | `rendition_failed` | 呈现版本失败[原因](#error-reasons)（如果有）。 |
| `errorMessage` | `string` | `rendition_failed` | 此文本提供了有关演绎版失败的更多详细信息（如果有）。 |

### 元数据 {#metadata}

| 属性 | 描述 |
|--------|-------------|
| `repo:size` | 演绎版的大小，以字节为单位。 |
| `repo:sha1` | 演绎版的sha1摘要。 |
| `dc:format` | 演绎版的 MIME 类型。 |
| `repo:encoding` | 演绎版的字符集编码（如果是基于文本的格式）。 |
| `tiff:ImageWidth` | 演绎版的宽度（像素）。 仅适用于图像演绎版。 |
| `tiff:ImageLength` | 演绎版的长度（以像素为单位）。 仅适用于图像演绎版。 |

### 错误原因 {#error-reasons}

| 原因 | 描述 |
|---------|-------------|
| `RenditionFormatUnsupported` | 给定的源不支持请求的演绎版格式。 |
| `SourceUnsupported` | 特定源不受支持，即使类型受支持也是如此。 |
| `SourceCorrupt` | 源数据已损坏。 它包含空文件。 |
| `RenditionTooLarge` | 无法使用`target`中提供的预签名URL上载节目。 实际演绎版大小在`repo:size`中作为元数据提供，客户端使用它重新处理此演绎版，并保留正确数量的预签名URL。 |
| `GenericError` | 任何其他意外错误。 |
