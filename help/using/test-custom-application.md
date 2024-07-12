---
title: 测试和调试 [!DNL Asset Compute Service] 自定义应用程序
description: 测试和调试 [!DNL Asset Compute Service] 自定义应用程序。
exl-id: c2534904-0a07-465e-acea-3cb578d3bc08
source-git-commit: c6f747ebd6d1b17834f1af0837609a148804f8a9
workflow-type: tm+mt
source-wordcount: '775'
ht-degree: 0%

---

# 测试和调试自定义应用程序 {#test-debug-custom-worker}

## 为自定义应用程序运行单元测试 {#test-custom-worker}

在您的计算机上安装[Docker桌面](https://www.docker.com/get-started)。 要测试自定义工作程序，请在应用程序的根目录下运行以下命令：

```bash
$ aio app test
```

<!-- TBD
To run tests for a custom application, run `aio asset-compute test-worker` command at the root of the custom application application.

Document interactively running `adobe-asset-compute` commands `test-worker` and `run-worker`.
-->

此命令运行自定义单元测试框架，用于Asset compute项目中的应用程序操作，如下所述。 它通过`package.json`文件中的配置挂接。 也可以使用JavaScript单元测试，例如Jest。 `aio app test`同时运行两者。

[aio-cli-plugin-asset-compute](https://github.com/adobe/aio-cli-plugin-asset-compute#install-as-local-devdependency)插件作为开发依赖项嵌入到自定义应用程序应用程序应用程序中，因此不需要将其安装在生成/测试系统中。

### 应用单元测试框架 {#unit-test-framework}

使用Asset compute应用程序单元测试框架，您无需编写任何代码即可测试应用程序。 它依赖于源文件的格式副本应用原理。 必须设置特定的文件和文件夹结构，以使用测试源文件、可选参数、预期呈现版本和自定义验证脚本定义测试用例。 默认情况下，会比较格式副本以保持字节相等。 此外，可以使用简单的JSON文件轻松模拟外部HTTP服务。

### 添加测试 {#add-tests}

测试应位于项目根级别的`test`文件夹内。 每个应用程序的测试用例应位于路径`test/asset-compute/<worker-name>`中，每个测试用例有一个文件夹：

```yaml
action/
manifest.yml
package.json
...
test/
  asset-compute/
    <worker-name>/
        <testcase1>/
            file.jpg
            params.json
            rendition.png
        <testcase2>/
            file.jpg
            params.json
            rendition.gif
            validate
        <testcase3>/
            file.jpg
            params.json
            rendition.png
            mock-adobe.com.json
            mock-console.adobe.io.json
```

请查看[自定义应用程序示例](https://github.com/adobe/asset-compute-example-workers/)以了解一些示例。 详细参考如下。

### 测试输出 {#test-output}

Adobe Developer App Builder应用程序根目录中的`build`目录包含自定义应用程序的详细测试结果和日志。 这些详细信息也显示在`aio app test`命令的输出中。

### 模拟外部服务 {#mock-external-services}

您可以通过为测试方案创建`mock-<HOST_NAME>.json`文件（HOST_NAME是您要模拟的特定主机），在操作中模拟外部服务调用。 示例用例是对S3进行单独调用的应用程序。 新的测试结构将如下所示：

```json
test/
  asset-compute/
    <worker-name>/
      <testcase3>/
        file.jpg
        params.json
        rendition.png
        mock-<HOST_NAME1>.json
        mock-<HOST_NAME2>.json
```

模拟文件是JSON格式的http响应。 有关详细信息，请参阅[此文档](https://www.mock-server.com/mock_server/creating_expectations.html)。 如果要模拟多个主机名，请定义多个`mock-<mocked-host>.json`文件。 以下是名为`mock-google.com.json`的`google.com`的示例模拟文件：

```json
[{
    "httpRequest": {
        "path": "/images/hello.txt"
        "method": "GET"
    },
    "httpResponse": {
        "statusCode": 200,
        "body": {
          "message": "hello world!"
        }
    }
}]
```

示例`worker-animal-pictures`包含与其交互的Wikimedia服务的[模拟文件](https://github.com/adobe/asset-compute-example-workers/blob/master/projects/worker-animal-pictures/test/asset-compute/worker-animal-pictures/simple-test/mock-upload.wikimedia.org.json)。

#### 跨测试用例共享文件 {#share-files-across-test-cases}

如果跨多个测试共享`file.*`、`params.json`或`validate`脚本，Adobe建议使用相对符号链接。 它们受Git支持。 请确保为您的共享文件指定一个唯一的名称，因为您可能具有不同的名称。 在下面的示例中，测试将几个共享文件与其自己的文件进行混合和匹配：

```json
tests/
    file-one.jpg
    params-resize.json
    params-crop.json
    validate-image-compare

    jpg-png-resize/
        file.jpg    - symlink: ../file-one.jpg
        params.json - symlink: ../params-resize.json
        rendition.png
        validate    - symlink: ../validate-image-compare

    jpg2-png-crop/
        file.jpg
        params.json - symlink: ../params-crop.json
        rendition.gif
        validate    - symlink: ../validate-image-compare

    jpg-gif-crop/
        file.jpg    - symlink: ../file-one.jpg
        params.json - symlink: ../params-crop.json
        rendition.gif
        validate
```

### 测试预期错误 {#test-unexpected-errors}

错误测试用例不应包含预期的`rendition.*`文件，应在`params.json`文件中定义预期的`errorReason`。

>[!NOTE]
>
>如果测试用例不包含预期的`rendition.*`文件，并且未在`params.json`文件中定义预期的`errorReason`，则假定它是包含任何`errorReason`的错误用例。

错误测试用例结构：

```json
<error_test_case>/
    file.jpg
    params.json
```

带有错误原因的参数文件：

```javascript
{
    "errorReason": "SourceUnsupported",
    // ... other params
}
```

查看[Asset compute错误原因](https://github.com/adobe/asset-compute-commons#error-reasons)的完整列表和说明。

## 调试自定义应用程序 {#debug-custom-worker}

以下步骤显示如何使用Visual Studio Code调试自定义应用程序。 它允许查看实时日志、点击断点和逐步执行代码，以及在每次激活时实时重新加载本地代码更改。

`aio`现成可自动完成其中的许多步骤。 转到[Adobe Developer App Builder文档](https://developer.adobe.com/app-builder/docs/getting_started/first_app)中的“调试应用程序”部分。 目前，以下步骤包括一个解决方法。

1. 从GitHub和可选的[ngrok](https://www.npmjs.com/package/ngrok)安装最新的[wskdebug](https://github.com/apache/openwhisk-wskdebug)。

   ```shell
   npm install -g @openwhisk/wskdebug
   npm install -g ngrok --unsafe-perm=true
   ```

1. 在JSON文件中添加用户设置。 它继续使用旧的Visual Studio代码调试器。 新环境的wskdebug `"debug.javascript.usePreview": false`存在[一些问题](https://github.com/apache/openwhisk-wskdebug/issues/74)。
1. 关闭通过`aio app run`打开的任何应用实例。
1. 使用`aio app deploy`部署最新的代码。
1. 仅使用`aio asset-compute devtool`运行Asset computeDevtool。 保持打开。
1. 在Visual Studio代码编辑器中，将以下调试配置添加到您的`launch.json`：

   ```json
   {
     "type": "node",
     "request": "launch",
     "name": "wskdebug worker",
     "runtimeExecutable": "wskdebug",
     "args": [
       "PROJECT-0.0.1/__secured_worker",           // Replace this with your ACTION NAME
       "${workspaceFolder}/actions/worker/index.js",
       "-l",
       "--ngrok"
     ],
     "localRoot": "${workspaceFolder}",
     "remoteRoot": "/code",
     "outputCapture": "std",
     "timeout": 30000
   }
   ```

   从`aio app deploy`的输出中获取`ACTION NAME`。

1. 从运行/调试配置中选择`wskdebug worker`并按播放图标。 等待它启动，直到它在&#x200B;**[!UICONTROL 调试控制台]**&#x200B;窗口中显示&#x200B;**[!UICONTROL 准备激活]**。

1. 在Devtool中单击&#x200B;**[!UICONTROL 运行]**。 您可以看到在Visual Studio代码编辑器中运行的操作，并且日志开始显示。

1. 在代码中设置断点。 然后再次运行，应该会命中。

任何代码更改都将实时加载，并在下次激活发生后立即生效。

>[!NOTE]
>
>自定义应用程序中的每个请求都存在两个激活。 第一个请求是一个Web操作，该操作在SDK代码中异步调用自身。 第二次激活是点击代码的激活。
