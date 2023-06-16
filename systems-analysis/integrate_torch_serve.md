# Integrate TorchServe
## 版本记录：
| 版本 | 修改时间 | 备注 |
| --- | --- | --- |
| 0.0.1 | 2023.05.30 | 初稿 |
| 0.0.2 | 2023.06.06 | 补充方案设计 |

## 需求分析：

- 编写 Torch Serve 插件，并将插件集成进 Torch Serve 中。
- 利用插件帮助 Dragonfly 完成文件分发功能。
## 目标：

- 实现插件的端口。
- 利用端口的 POST 请求实现文件的下载。
- 将下载后的模型文件注册到 Torch Serve 的模型列表中供后续调用。
## 技术架构：
### 设计思路时序图：
![image.png](https://cdn.nlark.com/yuque/0/2023/png/35887099/1685662028236-5d691069-9088-49d4-a954-9cd0a5f34af7.png#averageHue=%23f5f5f4&clientId=u9befa6d8-69e1-4&from=paste&height=366&id=ud7df57c8&originHeight=732&originWidth=922&originalType=binary&ratio=2&rotation=0&showTitle=false&size=67147&status=done&style=none&taskId=uc1da620b-0f67-4661-96f9-ac555f4f738&title=&width=461)
### 端点插件实现

1. API端点插件：

Torch Serve 的 serving-SDK 中提供一个名为 ModelServerEndpoint 的 JAVA 类。自定义其子类的 doPOST, doGET, doDELECT, doPUT 函数来实现从 Dragonfly 拉取和注册文件。将 Dragonfly 的配置参数，如访问密码等从环境变量里传入，而将与 Torch Serve 有关的参数从 config.properties 文件传入。
用户可以在启动 Torch Serve 后，利用自定义的端点来从 Dragonfly 远程拉取模型文件，存储在规定的 model_store 中，并对模型进行注册。API 端点插件的实现主要分为以下几步：

- 编写端点插件的 JAVA 类。
- 编写 gradle 文件，并将端口插件类打包为 jar 包。
- 设置 Dragonfly 相关的环境变量。
- 启动 Torch Serve，通过`torchserve --start --model-store <your-model-store-path> --plugins-path=<path-to-plugin-jars>`来初始化端点插件。
- 利用 REST API 调用自定义端点插件。
2. 端点函数实现：
```java
//文件名需要与类名一致
package org.pytorch.serve.plugins.s3;

import org.pytorch.serve.servingsdk.Context;
import org.pytorch.serve.servingsdk.ModelServerEndpoint;
import org.pytorch.serve.servingsdk.annotations.Endpoint;
import org.pytorch.serve.servingsdk.annotations.helpers.EndpointTypes;
import org.pytorch.serve.servingsdk.http.Request;
import org.pytorch.serve.servingsdk.http.Response;

import com.amazonaws.auth.AWSStaticCredentialsProvider;
import com.amazonaws.auth.BasicAWSCredentials;
import com.amazonaws.services.s3.AmazonS3;
import com.amazonaws.services.s3.AmazonS3ClientBuilder;
import com.amazonaws.services.s3.model.S3Object;

import java.io.IOException;
import java.io.InputStream;
import io.netty.handler.codec.http.QueryStringDecoder;
import java.nio.file.StandardCopyOption;
import java.nio.charset.StandardCharsets;
import java.nio.file.Files;
import java.nio.file.Paths;


//必要部分
@Endpoint(
    //端口的唯一标识符
    urlPattern = "S3Download",
    //端口种类
    endpointType = EndpointTypes.MANAGEMENT,
    description = "download file from s3.")
public class S3Download extends ModelServerEndpoint {
    //从环境变量中获取密钥
    private String awsAccessKey = System.getenv("AWS_ACCESS_KEY_ID");
    private String awsSecretKey = System.getenv("AWS_SECRET_ACCESS_KEY");
    private String regionName = System.getenv("AWS_DEFAULT_REGION");


    public void doPost(Request req, Response rsp, Context ctx) throws IOException {
        rsp.setStatus(200);


        // 初始化访问凭证和 S3 客户端
        BasicAWSCredentials awsCreds = new BasicAWSCredentials(awsAccessKey, awsSecretKey);

        AmazonS3 s3Client = AmazonS3ClientBuilder.standard()
        .withRegion(regionName)
        .withCredentials(new AWSStaticCredentialsProvider(awsCreds))
        .build();

        //文件名称和文件桶
        //返回list，而非string
        String bucketName = req.getParameter("bucket").get(0);
        String key = req.getParameter("key").get(0); 

        S3Object s3object = s3Client.getObject(bucketName, key);
        InputStream is = s3object.getObjectContent();

        Files.copy(is, Paths.get("/path/local_file"), StandardCopyOption.REPLACE_EXISTING);
        rsp.getOutputStream().write(("File downloaded from S3 successfully").getBytes(StandardCharsets.UTF_8));
    }
    
    public void doPut{
    }
    
    public void doGet{
    }
    
    public void doDelect{
    }

}

```

3. 端点（REST API）调用示例：
```
curl -X POST "http://localhost:8081/S3Download?bucket=11&key=2"
curl -X POST "http://localhost:8081/models?url=/path/to/your/model/model.mar&model_name=model_name&initial_workers=1&synchronous=true"
```

## 方案设计
### 目录结构
```bash
dragonfly-plugin
├── s3-endpoint
│   ├── build.gradle
│   └── src/main
│       ├── resources/META-INF/services
│       │   └── org.pytorch.serve.servingsdk.ModelServerEndpoint
│       └── java/org/pytorch/serve/plugins/s3endpoint
│         	├── S3Download.java
│         	└── Config.java
└── docs
    └── README.md
```
### 配置文件

1. Torch Serve 配置文件

Torch Serve 在两个地方会用到配置文件：1. 在初始化时可以通过`--ts-config`指向一个 config.properties 文件，支持 Torch Serve 和具体模型的相关配置。2. 在模型存档时利用`--config-file`指向一个 YAML 文件，只支持模型相关的配置。而与 AWS 相关的配置则是通过环境变量传入的。

2. 文件系统的 (Dragonfly, S3）的配置文件

上面提到的两种配置方法都可以实现模型的配置，但是模型启动或者存档时已经对模型完成了配置。如果想在调用插件时对模型文件的配置进行修改，需要重新定义相关配置。将文件系统和模型相关的配置文件放入一个json文件中，在插件调用时读取需要修改的配置。
```json
{
  // 文件系统相关
  "accessKeyId": "YOUR_ACCESS_KEY",
  "secretAccessKey": "YOUR_SECRET_KEY",
  "region": "us-west-2",
  "bucketName": "your-bucket-name",
  "key": "your-object-key",

  // 模型加载相关
  // 默认每次注册一个模型，且minWorker 为0（不分配工作线程）
  "load_model": "all",
  "minWorker": "1",
  
}
```
### 模型下载和注册

1. Torch Serve 模型下载和注册

Torch Serve 有两种注册模型的方法：1. 在初始化时通过配置文件或者命令行注册，这种方法可以批量注册，支持通过 URL 下载和注册, 但是不支持 S3 加密。2. 利用 Manage API 注册，这种方法支持 URL 下载和 S3 加密，但是一次只能注册一个，并且需要利用其他 Manage API 或在模型存档时 将 Worker 数量设置为大于0之后才能启动工作线程，进行模型预测。

当通过 Manage API 远程拉取加密的 S3 文件时，需要先设置`AWS_ACCESS_KEY_ID`,`AWS_SECRET_ACCESS_KEY`, `AWS_DEFAULT_REGION`三个环境变量，然后在 HttpUtiliy.java 通过编写 S3 的授权头部来下载和存储 S3 上的模型文件。总的来说 S3 的配置文件时通过环境变量传入，然后通过 S3 的授权头来下载文件。
```
$ 初始化时注册模型
torchserve --start --model-store model_store --models not-hot-dog=super-fancy-net.mar

$ Manage API 注册模型
curl -X POST  "http://localhost:8081/models?url=https://torchserve.pytorch.org/sse-test/squeezenet1_1.mar&s3_sse_kms=true"
```

2. 插件的模型注册

端口插件有两个重要的查询字符串，一个是 config 等于插件的配置文件地址，一个是 model 等于要加载的模型文件。为了和 Torch Serve 保持一致，model 也提供了3种加载方法（all，单个文件，多个文件）。同样 Torch Serve 一样，model 的参数可以覆盖配置文件参数。
```bash
curl -X PUT "http://localhost:8081/S3Download?config='../config.json&model=all"
```
```java
//包
package org.pytorch.serve.plugins.s3endpoint
//Torchserve相关
import org.pytorch.serve.servingsdk.Context;
import org.pytorch.serve.servingsdk.ModelServerEndpoint;
import org.pytorch.serve.servingsdk.annotations.Endpoint;
import org.pytorch.serve.servingsdk.annotations.helpers.EndpointTypes;
import org.pytorch.serve.servingsdk.http.Request;
import org.pytorch.serve.servingsdk.http.Response;
//S3相关
import software.amazon.awssdk.auth.credentials.AwsBasicCredentials;
import software.amazon.awssdk.regions.Region;
import software.amazon.awssdk.services.s3.S3Client;
import software.amazon.awssdk.services.s3.model.GetObjectRequest;

import com.fasterxml.jackson.databind.ObjectMapper;
import java.io.IOException;
import java.io.InputStream;
import io.netty.handler.codec.http.QueryStringDecoder;
import java.nio.file.StandardCopyOption;
import java.nio.charset.StandardCharsets;
import java.nio.file.Files;
import java.nio.file.Paths;


//必要部分
@Endpoint(
    //端口的唯一标识符
    urlPattern = "S3Download",
    //端口种类
    endpointType = EndpointTypes.MANAGEMENT,
    description = "download file from s3.")

public class S3Download extends ModelServerEndpoint {
    private final Map<String, String> config;

    
    private void initConfig(Request req) {
    	//如果 request 的查询字符串有 configPath 参数，读取配置文件进行赋值
        String configPath = req.getParameter("config").get(0);
        ObjectMapper mapper = new ObjectMapper();
        Map<String, String> config = mapper.readValue(new File(configPath), Map.class);
        //如果 request 的查询字符串没有 configPath 参数, 使用 Torch Serve 默认值
        this.config = config
    }

    
	@Override
    //从文件系统下载模型文件
    //同 Torch Serve 配置一致，支持三种状态下的模型初始化（文件夹下的全部文件，多个指定文件，单一指定文件）
    public void doPut(Request req, Response rsp, Context ctx) throws IOException {
        rsp.setStatus(200);
        initConfig(Request req);

        //建立S3客户端
        S3Client s3 = S3Client.builder()
        .region(Region.of(config.get("region")))
        .credentialsProvider(() -> AwsBasicCredentials.create(config.get("accessKeyId"), config.get("secretAccessKey")))
        .build();

        //通过查询字符串读取要下载的模型
        String models = req.getParameter("model").get(0);
        //如果未设置读取配置文件
        if (models == null || models.isEmpty()){
        	String models = config.get("load_model"))
        }

        //根据models的参数加载注册模型
        if ("all".equals(models)) {
            // 注册所有模型
        } else if (models.contains(",")) {
            // 注册多个模型
            String[] filenames = models.split(",");
        } else {
            // 注册一个模型
            String filename = models;
        }
        
        //返回response
    	String responseStr = "Download and register" + models + "successfully";
        rsp.getOutputStream().write(responseStr.getBytes(StandardCharsets.UTF_8));
        rsp.getOutputStream().flush();
    }

```
### 设计时序图
![image.png](https://cdn.nlark.com/yuque/0/2023/png/35887099/1686038926331-6bd38938-db86-4451-be84-865e599acffd.png#averageHue=%23f5f5f4&clientId=u9befa6d8-69e1-4&from=paste&height=355&id=u0fa85b4c&originHeight=710&originWidth=976&originalType=binary&ratio=2&rotation=0&showTitle=false&size=74719&status=done&style=none&taskId=u9553bb4e-f646-4dcb-af18-7918fcd8850&title=&width=488)
