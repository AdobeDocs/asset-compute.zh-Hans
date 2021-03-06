---
title: '"[!DNL Asset Compute Service] HTTP API"'
description: '"[!DNL Asset Compute Service] 用于创建自定义应用程序的HTTP API。”'
exl-id: 4b63fdf9-9c0d-4af7-839d-a95e07509750
source-git-commit: 93d3b407c8875888f03bec673d0a677a3205cfbb
workflow-type: tm+mt
source-wordcount: '2906'
ht-degree: 2%

---

# [!DNL Asset Compute Service] HTTP API {#asset-compute-http-api}

API的使用仅限于开发目的。 在开发自定义应用程序时，API将作为上下文提供。 [!DNL Adobe Experience Manager] as a [!DNL Cloud Service] 会使用API将处理信息传递到自定义应用程序。 有关更多信息，请参阅 [使用资产微服务和处理配置文件](https://experienceleague.adobe.com/docs/experience-manager-cloud-service/assets/manage/asset-microservices-configure-and-use.html).

>[!NOTE]
>
>[!DNL Asset Compute Service] 仅可用于 [!DNL Experience Manager] as a [!DNL Cloud Service].

的任何客户 [!DNL Asset Compute Service] HTTP API必须遵循以下高级流程：

1. 客户端设置为 [!DNL Adobe Developer Console] 项目。 每个单独的客户端（系统或环境）都需要其自己的单独项目来分隔事件数据流。

1. 客户端使用 [JWT（服务帐户）身份验证](https://www.adobe.io/authentication/auth-methods.html).

1. 客户端调用 [`/register`](#register) 检索日志URL的次数。

1. 客户端调用 [`/process`](#process-request) ，以生成演绎版。 调用是异步的。

1. 客户定期轮询日志以 [接收事件](#asynchronous-events). 当演绎版成功处理时，它会接收每个请求的演绎版的事件(`rendition_created` 事件类型)，或者`rendition_failed` 事件类型)。

的 [@adobe/asset-compute-client](https://github.com/adobe/asset-compute-client) 通过模块，可以轻松地在Node.js代码中使用API。

## 身份验证和授权 {#authentication-and-authorization}

所有API都需要访问令牌身份验证。 请求必须设置以下标头：

1. `Authorization` 带有不记名令牌的标头，技术帐户令牌，通过 [JWT交换](https://www.adobe.io/authentication/auth-methods.html) 从Adobe开发人员控制台项目。 的 [范围](#scopes) 下面介绍了。

<!-- TBD: Change the existing URL to a new path when a new path for docs is available. The current path contains master word that is not an inclusive term. Logged ticket in Adobe I/O's GitHub repo to get a new URL.
-->

1. `x-gw-ims-org-id` 标头（包含IMS组织ID）。

1. `x-api-key` 的客户端ID [!DNL Adobe Developers Console] 项目。

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

这些要求 [!DNL Adobe Developer Console] 要订阅的项目 `Asset Compute`, `I/O Events`和 `I/O Management API` 服务。 单个范围的划分是：

* 基本
   * 范围： `openid,AdobeID`

* asset compute
   * metascope: `asset_compute_meta`
   * 范围： `asset_compute,read_organizations`

* [!DNL Adobe I/O] 事件
   * metascope: `event_receiver_api`
   * 范围： `event_receiver,event_receiver_api`

* [!DNL Adobe I/O] 管理API
   * metascope: `ent_adobeio_sdk`
   * 范围： `adobeio_api,additional_info.roles,additional_info.projectedProductContext`

## 注册 {#register}

的每个客户 [!DNL Asset Compute service]  — 独特 [!DNL Adobe Developer Console] 订购服务的项目 — 必须 [注册](#register-request) 处理请求之前。 注册步骤会返回唯一事件日志，该日志用于从演绎版处理中检索异步事件。

在其生命周期结束时，客户端可以 [注销](#unregister-request).

### 注册请求 {#register-request}

此API调用可设置 [!DNL Asset Compute] 客户端，并提供事件日志URL。 这是幂等操作，每个客户端只需调用一次。 可以再次调用以检索日志URL。

| 参数 | 值 |
|--------------------------|------------------------------------------------------|
| 方法 | `POST` |
| 路径 | `/register` |
| 标题 `Authorization` | 全部 [授权相关标头](#authentication-and-authorization). |
| 标题 `x-request-id` | 可选，客户端可以为跨系统处理请求的唯一端到端标识符进行设置。 |
| 请求正文 | 必须为空。 |

### 注册响应 {#register-response}

| 参数 | 值 |
|-----------------------|------------------------------------------------------|
| MIME类型 | `application/json` |
| 标题 `X-Request-Id` | 与 `X-Request-Id` 请求标头或唯一生成的标头。 用于识别跨系统的请求和/或支持请求。 |
| 响应体 | 具有的JSON对象 `journal`, `ok` 和/或 `requestId` 字段。 |

HTTP状态代码为：

* **200次成功**:请求成功时。 它包含 `journal` 通过触发的异步处理结果通知的URL `/process` （作为事件类型） `rendition_created` 成功时，或 `rendition_failed` 失败时)。

   ```json
   {
       "ok": true,
       "journal": "https://api.adobe.io/events/organizations/xxxxx/integrations/xxxx/xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx",
       "requestId": "1234567890"
   }
   ```

* **401未授权**:请求无效时发生 [身份验证](#authentication-and-authorization). 示例可能是无效的访问令牌或无效的API密钥。

* **403禁止**:请求无效时发生 [授权](#authentication-and-authorization). 示例可能是有效的访问令牌，但Adobe开发人员控制台项目（技术帐户）未订阅所有必需的服务。

* **429请求过多**:在系统被此客户端重载时或其他情况下发生。 客户端应使用 [指数回退](https://en.wikipedia.org/wiki/Exponential_backoff). 尸体是空的。
* **4xx错误**:当存在任何其他客户端错误且注册失败时。 通常会返回诸如此类的JSON响应，但并不保证会出现所有错误：

   ```json
   {
       "ok": false,
       "requestId": "1234567890",
       "message": "error message"
   }
   ```

* **5xx错误**:在出现任何其他服务器端错误且注册失败时发生。 通常会返回诸如此类的JSON响应，但并不保证会出现所有错误：

   ```json
   {
       "ok": false,
       "requestId": "1234567890",
       "message": "error message"
   }
   ```

### 取消注册请求 {#unregister-request}

此API调用取消注册 [!DNL Asset Compute] 客户。 之后，将无法再调用 `/process`. 对未注册的客户端或尚未注册的客户端使用API调用会返回 `404` 错误。

| 参数 | 值 |
|--------------------------|------------------------------------------------------|
| 方法 | `POST` |
| 路径 | `/unregister` |
| 标题 `Authorization` | 全部 [授权相关标头](#authentication-and-authorization). |
| 标题 `x-request-id` | 可选，客户端可以为跨系统处理请求的唯一端到端标识符进行设置。 |
| 请求正文 | 空. |

### 取消注册响应 {#unregister-response}

| 参数 | 值 |
|-----------------------|------------------------------------------------------|
| MIME类型 | `application/json` |
| 标题 `X-Request-Id` | 与 `X-Request-Id` 请求标头或唯一生成的标头。 用于识别跨系统的请求和/或支持请求。 |
| 响应体 | 具有的JSON对象 `ok` 和 `requestId` 字段。 |

状态代码为：

* **200次成功**:在找到并删除注册和日记帐时发生。

   ```json
   {
       "ok": true,
       "requestId": "1234567890"
   }
   ```

* **401未授权**:请求无效时发生 [身份验证](#authentication-and-authorization). 示例可能是无效的访问令牌或无效的API密钥。

* **403禁止**:请求无效时发生 [授权](#authentication-and-authorization). 示例可能是有效的访问令牌，但Adobe开发人员控制台项目（技术帐户）未订阅所有必需的服务。

* **404未找到**:当给定凭据当前未注册时发生。

   ```json
   {
       "ok": true,
       "requestId": "1234567890"
   }
   ```

* **429请求过多**:在系统过载时发生。 客户端应使用 [指数回退](https://en.wikipedia.org/wiki/Exponential_backoff). 尸体是空的。

* **4xx错误**:发生任何其他客户端错误并取消注册失败的情况。 通常会返回诸如此类的JSON响应，但并不保证会出现所有错误：

   ```json
   {
       "ok": false,
       "requestId": "1234567890",
       "message": "error message"
   }
   ```

* **5xx错误**:在出现任何其他服务器端错误且注册失败时发生。 通常会返回诸如此类的JSON响应，但并不保证会出现所有错误：

   ```json
   {
       "ok": false,
       "requestId": "1234567890",
       "message": "error message"
   }
   ```

## 进程请求 {#process-request}

的 `process` 操作会根据请求中的说明提交一个作业，以将源资产转换为多个演绎版。 有关成功完成的通知（事件类型） `rendition_created`)或任何错误（事件类型） `rendition_failed`)会发送到事件日记帐，该日记帐必须使用 [/register](#register) 在生成任意数量之前 `/process` 请求。 错误设置的请求会立即失败，并出现400错误代码。

使用URL(如Amazon AWS S3预签名URL或Azure Blob Storage SAS URL)引用二进制文件，以便读取 `source` 资产(`GET` URL)和编写演绎版(`PUT` URL)。 客户负责生成这些预签名URL。

| 参数 | 值 |
|--------------------------|------------------------------------------------------|
| 方法 | `POST` |
| 路径 | `/process` |
| MIME类型 | `application/json` |
| 标题 `Authorization` | 全部 [授权相关标头](#authentication-and-authorization). |
| 标题 `x-request-id` | 可选，客户端可以为跨系统处理请求的唯一端到端标识符进行设置。 |
| 请求正文 | 必须采用如下所述的进程请求JSON格式。 它提供了有关要处理的资产以及要生成的演绎版的说明。 |

### 进程请求JSON {#process-request-json}

的请求正文 `/process` 是具有此高级架构的JSON对象：

```json
{
    "source": "",
    "renditions" : []
}
```

可用字段包括：

| 名称 | 类型 | 描述 | 示例 |
|--------------|----------|-------------|---------|
| `source` | `string` | 要处理的源资产的URL。 基于请求的呈现格式(例如， `fmt=zip`)。 | `"http://example.com/image.jpg"` |
| `source` | `object` | 描述要处理的源资产。 请参阅 [源对象字段](#source-object-fields) 下。 基于请求的呈现格式(例如， `fmt=zip`)。 | `{"url": "http://example.com/image.jpg", "mimeType": "image/jpeg" }` |
| `renditions` | `array` | 要从源文件生成的演绎版。 每个呈现对象支持 [再现指令](#rendition-instructions). 必填. | `[{ "target": "https://....", "fmt": "png" }]` |

的 `source` 可以 `<string>` 视为URL，或者它可以是 `<object>` 添加其他字段。 以下变体相似：

```json
"source": "http://example.com/image.jpg"
```

```json
"source": {
    "url": "http://example.com/image.jpg"
}
```

### 源对象字段 {#source-object-fields}

| 名称 | 类型 | 描述 | 示例 |
|-----------|----------|-------------|---------|
| `url` | `string` | 要处理的源资产的URL。 必填. | `"http://example.com/image.jpg"` |
| `name` | `string` | 源资产文件名。 如果未检测到MIME类型，则可以使用名称中的文件扩展名。 在的URL路径或文件名中，优先于文件名 `content-disposition` 二进制资源的标头。 默认为“file”。 | `"image.jpg"` |
| `size` | `number` | 源资产文件大小（以字节为单位）。 优先于 `content-length` 二进制资源的标头。 | `10234` |
| `mimetype` | `string` | 源资产文件MIME类型。 优先于 `content-type` 二进制资源的标头。 | `"image/jpeg"` |

### 结束 `process` 请求示例 {#complete-process-request-example}

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

的 `/process` 请求会根据基本请求验证立即返回，并出现成功或失败。 实际的资产处理是异步进行的。

| 参数 | 值 |
|-----------------------|------------------------------------------------------|
| MIME类型 | `application/json` |
| 标题 `X-Request-Id` | 与 `X-Request-Id` 请求标头或唯一生成的标头。 用于识别跨系统的请求和/或支持请求。 |
| 响应体 | 具有的JSON对象 `ok` 和 `requestId` 字段。 |

状态代码：

* **200次成功**:请求是否成功提交。 响应JSON包含 `"ok": true`:

   ```json
   {
       "ok": true,
       "requestId": "1234567890"
   }
   ```

* **400请求无效**:如果请求的格式不正确，例如请求JSON中缺少必填字段。 响应JSON包含 `"ok": false`:

   ```json
   {
       "ok": false,
       "requestId": "1234567890",
       "message": "error message"
   }
   ```

* **401未授权**:请求无效时 [身份验证](#authentication-and-authorization). 示例可能是无效的访问令牌或无效的API密钥。
* **403禁止**:请求无效时 [授权](#authentication-and-authorization). 示例可能是有效的访问令牌，但Adobe开发人员控制台项目（技术帐户）未订阅所有必需的服务。
* **429请求过多**:当系统由此客户端或通常的客户端过载时。 客户端可以使用 [指数回退](https://en.wikipedia.org/wiki/Exponential_backoff). 尸体是空的。
* **4xx错误**:出现任何其他客户端错误时。 通常会返回诸如此类的JSON响应，但并不保证会出现所有错误：

   ```json
   {
       "ok": false,
       "requestId": "1234567890",
       "message": "error message"
   }
   ```

* **5xx错误**:当存在任何其他服务器端错误时。 通常会返回诸如此类的JSON响应，但并不保证会出现所有错误：

   ```json
   {
       "ok": false,
       "requestId": "1234567890",
       "message": "error message"
   }
   ```

大多数客户端可能倾向于使用 [指数回退](https://en.wikipedia.org/wiki/Exponential_backoff) 在任何错误 *除外* 配置问题（如401或403）或无效请求（如400）。 除了通过429个响应来限制正常速率外，临时服务中断或限制可能会导致5xx错误。 建议在一段时间后重试。

所有JSON响应（如果存在）都包含 `requestId` 与 `X-Request-Id` 标题。 建议从标头中读取，因为它始终存在。 的 `requestId` 在与处理请求相关的所有事件中也返回为 `requestId`. 客户端不得对此字符串的格式作任何假设，因为它是不透明的字符串标识符。

## 选择加入后处理 {#opt-in-to-post-processing}

的 [asset computeSDK](https://github.com/adobe/asset-compute-sdk) 支持一组基本的图像后处理选项。 通过设置字段，自定义工作程序可以明确地选择启用后处理 `postProcess` 在演绎版对象上 `true`.

支持的用例包括：

* 将呈现内容裁剪为限制由crop.w、crop.h、crop.x和crop.y定义的矩形。它由 `instructions.crop` 在呈现版本对象中。
* 使用宽度、高度或两者调整图像大小。 它由 `instructions.width` 和 `instructions.height` 在呈现版本对象中。 要仅使用宽度或高度调整大小，请仅设置一个值。 计算服务可节省宽高比。
* 设置JPEG图像的质量。 它由 `instructions.quality` 在呈现版本对象中。 最佳质量由 `100` 值越小，质量越低。
* 创建隔行扫描图像。 它由 `instructions.interlace` 在呈现版本对象中。
* 设置DPI，通过调整应用于像素的比例来调整呈现大小，以便桌面发布。 它由 `instructions.dpi` 用于更改dpi分辨率的格式副本对象。 但是，要调整图像大小以使其以不同的分辨率大小相同，请使用 `convertToDpi` 说明。
* 调整图像大小，使其呈现的宽度或高度在指定目标分辨率(DPI)下保持与原始图像相同。 它由 `instructions.convertToDpi` 在呈现版本对象中。

## 为资源添加水印 {#add-watermark}

的 [asset computeSDK](https://github.com/adobe/asset-compute-sdk) 支持向PNG、JPEG、TIFF和GIF图像文件添加水印。 水印将按照 `watermark` 对象。

在再现后处理过程中会添加水印。 要对资产添加水印，自定义工作程序 [opt到后处理](#opt-in-to-post-processing) 通过设置字段 `postProcess` 在演绎版对象上 `true`. 如果工作程序没有选择加入，则不会应用水印，即使在请求的呈现对象上设置了水印对象也是如此。

## 再现说明 {#rendition-instructions}

这些选项适用于 `renditions` 阵列 [/process](#process-request).

### 常用字段 {#common-fields}

| 名称 | 类型 | 描述 | 示例 |
|-------------------|----------|-------------|---------|
| `fmt` | `string` | 演绎版目标格式，也可以 `text` 用于文本提取和 `xmp` 将XMP元数据提取为xml。 请参阅 [支持的格式](https://experienceleague.adobe.com/docs/experience-manager-cloud-service/assets/file-format-support.html) | `png` |
| `worker` | `string` | 的URL [自定义应用程序](develop-custom-application.md). 必须是 `https://` URL。 如果存在此字段，则演绎版由自定义应用程序创建。 然后，任何其他集的呈现版本字段将用于自定义应用程序。 | `"https://1234.adobeioruntime.net`<br>`/api/v1/web`<br>`/example-custom-worker-master/worker"` |
| `target` | `string` | 应使用HTTPPUT将生成的演绎版上传到的URL。 | `http://w.com/img.jpg` |
| `target` | `object` | 生成的演绎版的多部分预签名URL上传信息。 这是为 [AEM/Oak直接二进制上传](https://jackrabbit.apache.org/oak/docs/features/direct-binary-access.html) 此 [多部分上传行为](https://jackrabbit.apache.org/oak/docs/apidocs/org/apache/jackrabbit/api/binary/BinaryUpload.html).<br>字段:<ul><li>`urls`:字符串数组，每个预签名的部件URL对应一个</li><li>`minPartSize`:用于一部分的最小大小= url</li><li>`maxPartSize`:一部分使用的最大大小= url</li></ul> | `{ "urls": [ "https://part1...", "https://part2..." ], "minPartSize": 10000, "maxPartSize": 100000 }` |
| `userData` | `object` | 由客户端控制并按原样传递到呈现版本事件的可选保留空间。 允许客户端添加自定义信息以标识呈现事件。 在自定义应用程序中不得修改或依赖，因为客户可以随时更改。 | `{ ... }` |

### 演绎版特定字段 {#rendition-specific-fields}

有关当前支持的文件格式的列表，请参阅 [支持的文件格式](https://experienceleague.adobe.com/docs/experience-manager-cloud-service/assets/file-format-support.html).

| 名称 | 类型 | 描述 | 示例 |
|-------------------|----------|-------------|---------|
| `*` | `*` | 可添加高级自定义字段，以 [自定义应用程序](develop-custom-application.md) 明白。 |  |
| `embedBinaryLimit` | `number` 字节 | 如果设置了此值，并且演绎版的文件大小小于此值，则演绎版将嵌入在演绎版生成完成后发送的事件中。 允许嵌入的最大大小为32 KB（32 x 1024字节）。 如果演绎版的大小大于 `embedBinaryLimit` 限制，则该数据库将放置在云存储中的某个位置，且不会嵌入到事件中。 | `3276` |
| `width` | `number` | 宽度（以像素为单位）. 仅用于图像呈现。 | `200` |
| `height` | `number` | 高度（以像素为单位）. 仅用于图像呈现。 | `200` |
|  |  | 在以下情况下，始终保持宽高比： <ul> <li> 两者兼有 `width` 和 `height` ，则图像在保持宽高比的同时适合大小 </li><li> 仅 `width` 仅 `height` 指定，则生成的图像在保持宽高比的同时使用相应的尺寸</li><li> 如果两者都不 `width` nor `height` ，则会使用原始图像像素大小。 取决于源类型。 对于某些格式(如PDF文件)，使用默认大小。 可以有最大大小限制。</li></ul> |  |
| `quality` | `number` | 在 `1` to `100`. 仅适用于图像演绎版。 | `90` |
| `xmp` | `string` | 仅由XMP元数据写回使用，它是base64编码的XMP，用于写回到指定的呈现版本。 |  |
| `interlace` | `bool` | 通过将隔行PNG或GIF或逐行JPEG设置为 `true`. 它对其他文件格式没有影响。 |  |
| `jpegSize` | `number` | JPEG文件的大小（以字节为单位）。 它会覆盖任何 `quality` 设置。 对其他格式没有影响。 |  |
| `dpi` | `number` 或 `object` | 设置x和y DPI。 为简便起见，还可将其设置为用于x和y的单个数字。它对图像本身没有影响。 | `96` 或 `{ xdpi: 96, ydpi: 96 }` |
| `convertToDpi` | `number` 或 `object` | x和y DPI在保持物理大小的同时对值进行重新采样。 为简便起见，还可将其设置为用于x和y的单个数字。 | `96` 或 `{ xdpi: 96, ydpi: 96 }` |
| `files` | `array` | 要包含在ZIP存档中的文件列表(`fmt=zip`)。 每个条目可以是URL字符串，也可以是具有以下字段的对象：<ul><li>`url`:要下载文件的URL</li><li>`path`:将文件存储在ZIP中此路径下</li></ul> | `[{ "url": "https://host/asset.jpg", "path": "folder/location/asset.jpg" }]` |
| `duplicate` | `string` | ZIP存档处理重复(`fmt=zip`)。 默认情况下，存储在ZIP中同一路径下的多个文件会生成错误。 设置 `duplicate` to `ignore` 只会导致存储第一个资产，而忽略其余资产。 | `ignore` |
| `watermark` | `object` | 包含有关 [水印](#watermark-specific-fields). |  |

### 特定于水印的字段 {#watermark-specific-fields}

PNG格式用作水印。

| 名称 | 类型 | 描述 | 示例 |
|-------------------|----------|-------------|---------|
| `scale` | `number` | 水印的比例，介于 `0.0` 和 `1.0`. `1.0` 即水印具有其原始比例(1:1)，且值越低，水印大小就越小。 | 值 `0.5` 是原来大小的一半。 |
| `image` | `url` | 要用于水印的PNG文件的URL。 |  |

## 异步事件 {#asynchronous-events}

演绎版处理完成后或发生错误时，会向 [[!DNL Adobe I/O] 事件日记帐](https://www.adobe.io/apis/experienceplatform/events/documentation.html#!adobedocs/adobeio-events/master/intro/journaling_api.md). 客户必须监听通过提供的日志URL [/register](#register). 日志响应包括 `event` 由每个事件的一个对象组成的数组，其中 `event` 字段包含实际的事件有效负载。

的 [!DNL Adobe I/O] 适用于 [!DNL Asset Compute Service] is `asset_compute`. 日记帐仅自动订阅此事件类型，无需根据 [!DNL Adobe I/O] 事件类型。 在 `type` 事件的属性。

### 事件类型 {#event-types}

| Event | 描述 |
|---------------------|-------------|
| `rendition_created` | 已为已成功处理且已上传的每个演绎版发送。 |
| `rendition_failed` | 已发送给处理或上传失败的每个演绎版。 |

### 事件属性 {#event-attributes}

| 属性 | 类型 | 事件 | 描述 |
|-------------|----------|---------------|-------------|
| `date` | `string` | `*` | 在简化的扩展中发送事件的时间戳 [ISO-8601](https://en.wikipedia.org/wiki/ISO_8601) 格式，由JavaScript定义 [Date.toISOString()](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Date/toISOString). |
| `requestId` | `string` | `*` | 原始请求的请求ID `/process`与相同 `X-Request-Id` 标题。 |
| `source` | `object` | `*` | 的 `source` 的 `/process` 请求。 |
| `userData` | `object` | `*` | 的 `userData` 的 `/process` 请求（如果已设置）。 |
| `rendition` | `object` | `rendition_*` | 传入的相应呈现对象 `/process`. |
| `metadata` | `object` | `rendition_created` | 的 [元数据](#metadata) 演绎版的属性。 |
| `errorReason` | `string` | `rendition_failed` | 演绎版失败 [原因](#error-reasons) 如果有的话。 |
| `errorMessage` | `string` | `rendition_failed` | 文本提供了有关演绎版失败（如果有）的更多详细信息。 |

### 元数据 {#metadata}

| 属性 | 描述 |
|--------|-------------|
| `repo:size` | 演绎版的大小（以字节为单位）。 |
| `repo:sha1` | 呈现版本的sha1摘要。 |
| `dc:format` | 演绎版的MIME类型。 |
| `repo:encoding` | 演绎版的字符集编码，以防其为基于文本的格式。 |
| `tiff:ImageWidth` | 演绎版的宽度（以像素为单位）。 仅适用于图像呈现。 |
| `tiff:ImageLength` | 演绎版的长度（以像素为单位）。 仅适用于图像呈现。 |

### 错误原因 {#error-reasons}

| 原因 | 描述 |
|---------|-------------|
| `RenditionFormatUnsupported` | 给定源不支持请求的格式副本格式。 |
| `SourceUnsupported` | 即使支持类型，特定源也不受支持。 |
| `SourceCorrupt` | 源数据已损坏。 包括空文件。 |
| `RenditionTooLarge` | 无法使用 `target`. 实际演绎版大小可在 `repo:size` 和可供客户端使用，以使用正确数量的预签名URL重新处理此演绎版。 |
| `GenericError` | 任何其他意外错误。 |
