# Triton Server
## 一个简单的 Use Case

- 创建一个 Model Repo 。
```bash
./fetch_models.sh  ->  model_repository  ## 通过一个脚本文件将远程的模型仓库拉到本地
```

- 跑一个 Triton Server 。
```bash
docker run --gpus=1 --rm --net=host -v ${PWD}/model_repository:/models nvcr.io/nvidia/tritonserver:23.04-py3 tritonserver --model-repository=/models
```

- 向 Triton Server 发送推理请求并获得响应。
```bash
docker run -it --rm --net=host nvcr.io/nvidia/tritonserver:23.04-py3-sdk
/workspace/install/bin/image_client -m densenet_onnx -c 3 -s INCEPTION /workspace/images/mug.jpg
```

## 框架概览
![image.png](https://cdn.nlark.com/yuque/0/2023/png/22094377/1683545285766-fcf55744-9ba4-4861-af53-42ff48563590.png#averageHue=%23f8f8f8&clientId=u26f92c63-a36e-4&from=paste&id=uc4a83b0f&originHeight=874&originWidth=691&originalType=url&ratio=1&rotation=0&showTitle=false&size=211854&status=done&style=none&taskId=uf226089b-814a-4895-bff8-1affe41c968&title=)

- model repository 是一个基于文件系统的模型仓库，Triton Server 的作用就是使仓库中的模型可应用于推理任务。
- 客户端通过 HTTP/REST、GRPC 或者 C AP I的方式向 Triton Server 发送推理请求，请求被路由到合适的 per-model scheduler 。
- Triton 实现了多个调度和批处理算法，这些算法可以按模型的粒度进行配置。每个模型的调度器批量地执行推理请求，然后将请求传递到与模型类型对应的后端。
- 后端处理批量请求的输入并返回输出。
- Triton支持一个后端 C API，允许 Triton 扩展新的功能，如自定义预处理和后处理操作，甚至是一个新的深度学习框架。
- Triton 所服务的模型可以通过 [model management API](https://github.com/triton-inference-server/server/blob/main/docs/user_guide/model_management.md) 进行查询和控制。
## Model Repository
模型仓库路径在启动 Triton 时通过使用 --model-repository 选项来指定。可以多次指定 --model-repository 选项，以包含来自多个模型仓库的模型。
```bash
$ tritonserver --model-repository=<model-repository-path>
```
### 模型仓库布局
模型仓库的文件布局如下：
```bash
  <model-repository-path>/
    <model-name>/
      [config.pbtxt]
      [<output-labels-file> ...]
      <version>/
        <model-definition-file>
      <version>/
        <model-definition-file>
      ...
    <model-name>/
      [config.pbtxt]
      [<output-labels-file> ...]
      <version>/
        <model-definition-file>
      <version>/
        <model-definition-file>
      ...
    ...
```
config.pbtxt 文件是必备的，它描述了模型的基本配置。除此之外，每个文件夹至少包含一个子文件夹来表示模型的版本，子文件夹中必须有模型对应的框架后端(如 TensorRT, PyTorch, ONNX, OpenVINO 和TensorFlow)所规定的模型文件。
### 模型仓库位置
模型仓库的位置可以是本地文件系统或者云存储。Triton 支持的云存储有 Google Cloud Storage、S3 和 Azure Storage，可以通过设置证书文件来访问云存储中的 model repository 。
![image.png](https://cdn.nlark.com/yuque/0/2023/png/22094377/1683693385628-eaa48088-9f2c-496b-a3b5-3cff5245fb77.png#averageHue=%23757676&clientId=ue1a329bd-0c62-4&from=paste&height=50&id=u183df1e6&originHeight=50&originWidth=402&originalType=binary&ratio=1&rotation=0&showTitle=false&size=3640&status=done&style=none&taskId=u3b544925-fcd7-4bc7-ae3f-a89722306be&title=&width=402)
```bash
export TRITON_CLOUD_CREDENTIAL_PATH="cloud_credential.json"

“cloud_credential.json”:
{
  "gs": {
    "": "PATH_TO_GOOGLE_APPLICATION_CREDENTIALS",
    "gs://gcs-bucket-002": "PATH_TO_GOOGLE_APPLICATION_CREDENTIALS_2"
  },
  "s3": {
    "": {
      "secret_key": "AWS_SECRET_ACCESS_KEY",
      "key_id": "AWS_ACCESS_KEY_ID",
      "region": "AWS_DEFAULT_REGION",
      "session_token": "",
      "profile": ""
    },
    "s3://s3-bucket-002": {
      "secret_key": "AWS_SECRET_ACCESS_KEY_2",
      "key_id": "AWS_ACCESS_KEY_ID_2",
      "region": "AWS_DEFAULT_REGION_2",
      "session_token": "AWS_SESSION_TOKEN_2",
      "profile": "AWS_PROFILE_2"
    }
  },
  "as": {
    "": {
      "account_str": "AZURE_STORAGE_ACCOUNT",
      "account_key": "AZURE_STORAGE_KEY"
    },
    "as://Account-002/Container": {
      "account_str": "",
      "account_key": ""
    }
  }
}
```
### 模型文件
每个模型版本子目录的内容由模型的类型和支持模型的后端需求决定：

| ### TensorRT Models
 | model.plan |
| --- | --- |
| ### ONNX Models
 | model.onnx |
| ### TorchScript Models
 | model.pt |
| ### TensorFlow Models
 | model.graphdef |
| ### OpenVINO Models
 | model.xml / model.bin |
| ### Python Models
 | model.py |
| ### DALI Models
 | model.dali |

### API
```
## Index
$repository_index_request =
{
  "ready" : $boolean #optional,
}

$repository_index_response =
[
  {
    "name" : $string,
    "version" : $string #optional,
    "state" : $string,
    "reason" : $string
  },
  …
]

$repository_index_error_response =
{
  "error": $string
}

## Load
$repository_load_request =
{
  "parameters" : $parameters #optional
}

$repository_load_error_response =
{
  "error": $string
}

## Unload
$repository_unload_request =
{
  "parameters" : $parameters #optional
}

$repository_unload_error_response =
{
  "error": $string
}
```
```
## Index
message RepositoryIndexRequest
{
  // The name of the repository. If empty the index is returned
  // for all repositories.
  string repository_name = 1;

  // If true return only models currently ready for inferencing.
  bool ready = 2;
}

message RepositoryIndexResponse
{
  // Index entry for a model.
  message ModelIndex {
    // The name of the model.
    string name = 1;

    // The version of the model.
    string version = 2;

    // The state of the model.
    string state = 3;

    // The reason, if any, that the model is in the given state.
    string reason = 4;
  }

  // An index entry for each model.
  repeated ModelIndex models = 1;
}

## Load
message RepositoryModelLoadRequest
{
  // The name of the repository to load from. If empty the model
  // is loaded from any repository.
  string repository_name = 1;

  // The name of the model to load, or reload.
  string model_name = 2;

  // Optional parameters.
  map<string, ModelRepositoryParameter> parameters = 3;
}

message RepositoryModelLoadResponse
{
}

## Unload
message RepositoryModelUnloadRequest
{
  // The name of the repository from which the model was originally
  // loaded. If empty the repository is not considered.
  string repository_name = 1;

  // The name of the model to unload.
  string model_name = 2;

  // Optional parameters.
  map<string, ModelRepositoryParameter> parameters = 3;
}

message RepositoryModelUnloadResponse
{
}
```
## Model Management
控制模式决定了 Triton 如何处理对于模型仓库的更改，Triton 通过三种控制模式来控制模型的加载和卸载：
### None
Triton 在启动阶段加载仓库中的所有模型，后续运行阶段对仓库的所有更改将被 Triton 忽略，通过 [model control protocol](https://docs.nvidia.com/deeplearning/triton-inference-server/user-guide/docs/protocol/extension_model_repository.html) 对 Triton 的 load 和 unload 请求将会返回一个错误的响应。
### Explicit
Triton 在启动阶段通过 --load-model 指令明确指定要加载的模型，在启动之后，所有的模型 load 和 unload 操作必须通过使用 [model control protocol](https://docs.nvidia.com/deeplearning/triton-inference-server/user-guide/docs/protocol/extension_model_repository.html) 显式地启动。
### Poll
Triton 在启动阶段将尝试加载模型仓库中的所有模型，后续运行阶段对仓库的更改将被检测到，Triton 将根据这些更改尝试加载和卸载模型。Triton 周期性地检测仓库是否发生更改，可以通过 --repository-poll-secs 来调整检测周期。
## Triton 配置
### 命令行配置
#### Log 相关
Triton 提供四种等级的 Log：info warning error verbose，每一个选项可以分别选择是否打开。

- --log-verbose <integer>

表示是否启用 verbose logging，如果为 0 则表示禁用，>= 1 表示启用。具体的，int 类型的 level 值表明	日志记录的级别，小于等于所指定级别的日志将被输出。如下所示：
![image.png](https://cdn.nlark.com/yuque/0/2023/png/22094377/1684912296739-23b5fba8-05b3-4654-97c2-093f8db17429.png#averageHue=%232f2921&clientId=u978888e7-226c-4&from=paste&height=78&id=u43769a1c&originHeight=78&originWidth=1045&originalType=binary&ratio=1&rotation=0&showTitle=false&size=14833&status=done&style=none&taskId=u30da80c2-db92-4272-890b-3901dbcd1db&title=&width=1045)
![image.png](https://cdn.nlark.com/yuque/0/2023/png/22094377/1684912318934-5f231532-96d0-4f43-a751-48093b7f753c.png#averageHue=%23362d21&clientId=u978888e7-226c-4&from=paste&height=58&id=u559cdddb&originHeight=58&originWidth=974&originalType=binary&ratio=1&rotation=0&showTitle=false&size=15203&status=done&style=none&taskId=u50be584a-d51e-4e66-a636-260759854dd&title=&width=974)

- --log-info <boolean>
- --log-warning <boolean>
- --log-error <boolean>
#### 模型相关

- --model-store 和 --model-repository 一样，指定模型仓库地址，可以多次指定。
- --backend-directory: 指定 Backend 的位置。

实现自定义后端的动态库的存放目录，默认为 /opt/tritonserver/backend。

- --repoagent-directory: 指定 Agent 的位置。

实现自定义 repoagent 的动态库的存放目录，默认为 /opt/tritonserver/repoagents。

- --strict-model-config: true 表示一定要配置文件，false 则表示可以尝试自动填充。
- --model-control-mode: none, poll, explicit 三种。
- --repository-poll-secs: 轮询时长。
- --load-model: 配合 explicit 使用，指定启动的模型。
- --backend-config <<string>,<string>=<string>>: 给某个 Backend 加上特定的选项。

指定一个特定的后端，对该后端的配置进行设置，使用格式为：
```cpp
--backend-config=<backend_name>,<setting>=<value>
```
其中 backend_name 为后端的名字，如 tensorrt 等。
![image.png](https://cdn.nlark.com/yuque/0/2023/png/22094377/1684913076685-89b04d52-7a77-4e13-a955-1c6dd4220b4d.png#averageHue=%232a2825&clientId=u978888e7-226c-4&from=paste&height=46&id=u53f71c2d&originHeight=46&originWidth=1032&originalType=binary&ratio=1&rotation=0&showTitle=false&size=19202&status=done&style=none&taskId=u0564d68b-c2b8-436e-89fb-84b8503a40c&title=&width=1032)
#### 服务相关

- --id: 指定服务器标识符。
- --strict-readiness: true 表示服务器和模型启动完毕才可以访问 /v2/health/ready，false 表示服务器启动即可。
- --allow-http: 开启 HTTP 服务器。
- --allow-grpc: 开启 GRPC 服务器。
- --allow-sagemaker: 监听 Sagemaker 的请求。

AWS SageMaker Neo 是一种服务，可将训练好的机器学习模型编译为预测流水线，并具有加速推理和优化模型内存使用的能力。在 Triton Server 中，可以通过 --allow-sagemaker 参数开启对 SageMaker Neo 模型的支持，从而使得用户可以将 SageMaker Neo 编译后的模型部署到 Triton Server 实例上进行在线推理。

- --allow-metrics: 开启端口提供 Prometheus 格式的使用数据。
- --allow-gpu-metrics: 提供 GPU 相关的数据。

上面每一种 allow，都有配套的其他选项，比如 http 可以设置处理线程数量，grpc 可设置是否开启 ssl。
记录一条请求在 Triton 中执行的整个过程，记录接受请求，请求处理，排队，计算，发送请求等的时间戳。
#### Trace 相关

- --trace-file: 输出的位置。
- --trace-level: 等级，max 会记录更多信息，比如将计算拆解成输入、输出、推理等部分。
- --trace-rate: 频率，比如多少个请求采样一次。
#### CUDA 相关

- --pinned-memory-pool-byte-size: 锁页内存的大小，可以加速 host 和 device 的数据传输，默认 256MB。

主机端常规方式分配的内存（用new、malloc等方式）都是可分页（pageable）的，可分页内存在分配后是可能被操作系统移动的，GPU端无法获知操作系统是否正在移动。因此，当从可分页内存传输数据到设备内存时，CUDA驱动程序首先分配临时页面锁定的主机内存，将可分页内存复制到页面锁定内存中，然后再从页面锁定内存传输到设备内存。为了加快数据传输，可以在主机端直接分配锁页内存。锁页内存是主机端一块固定的物理内存，它不能被操作系统移动，不参与虚拟内存相关的交换操作。GPU知道锁页内存的物理地址，可以通过“直接内存访问（Direct Memory Access，DMA）”技术直接在主机和GPU之间复制数据，传输仅一次，效率更高。
![image.png](https://cdn.nlark.com/yuque/0/2023/png/22094377/1684914288305-aed35665-0d0e-4469-939a-298fe4226961.png#averageHue=%23f5fcfb&clientId=u978888e7-226c-4&from=paste&height=278&id=uef81c22f&originHeight=278&originWidth=707&originalType=binary&ratio=1&rotation=0&showTitle=false&size=77903&status=done&style=none&taskId=u8f9f8be9-b8c2-465c-9346-8070eb3c626&title=&width=707)

- --cuda-memory-pool-byte-size <<integer>:<integer>>: 第一个数字为 GPU 卡号，第二个数字显存大小，默认 64MB。主要作用是为指定的 GPU 分配指定大小的 CUDA 内存池。
- --min-supported-compute-capability <float>: 最低支持的 CUDA 计算能力。不支持 "该计算能力" 的GPU将不会被服务器使用。
#### 其他

- --exit-on-error: 发送错误的时候退出。
- --exit-timeout-secs <integer>: 当退出服务器的时候，有的请求还没处理完，这个选项指定了超时时间。
- --rate-limit 和 --rate-limit-resource: 

从效果上来说，Rate Limiter 的作用是限制了请求分发到模型实例上。从实现上来说，Rate Limiter 引入了 “Resource” 的概念，表示一个模型实例需要的资源，当系统中存在足够的资源，这个模型就会执行。如果资源不够，那么一个请求需要等待其他模型实例释放资源。最终的表现就是好像限制了速度一样。
其中，--rate-limit 设置了 rate limiting 的格式，具体有两种，默认为 off，即不对请求分发做速率上的限制，即来即服务。execution_count 模式则根据对应配置文件上的信息，当资源满足要求时予以服务。
```bash
--rate-limit-resource=R1:10 --rate-limit-resource=R2:5:0 --rate-limit-resource=R2:8:1 --rate-limit-resource=R3:2

GLOBAL: [R3: 2]
device0: [R1: 10, R2: 5]
device1: [R1: 10, R2: 8]

  instance_group [
    {
      count: 2
      kind: KIND_CPU
      rate_limiter {
        resources [
          {
            name: "R1"
            count: 10
          },
          {
            name: "R2"
            count: 5
          },
          {
            name: "R3"
            global: True
            count: 10
          }
        ] 
        priority: 1
      }
    }
  ]

```

- --response-cache-byte-size <integer>: 响应缓存的大小。·

已被弃用，现在使用 --cache-config <<string>,<string>=<string>>，如 --cache-config redis,size=1048576，指定缓存的方式以及大小，缓存方式有 redis 和 local 两种。

- --buffer-manager-thread-count: 指定线程数量，可以加速在输入输出 Tensor 上的操作。
### 模型配置文件
模型配置文件位于每个模型文件夹下的 config.pbtxt 中。
#### 为什么使用 pbtxt 作为配置文件的格式？
 Protocol Buffer（protobuf） 是谷歌开源的一种数据存储语言，每一个文件以 .proto 为后缀，它不依赖特定的语言与平台，扩展性极强。相较于 json、xml 等格式的数据占用存储更少，更加简洁高效。TensorFlow 内部的数据存储（比如 .pb 和 .ckpt 等）基本都使用 protobuf 格式。
Triton 在 model_config.proto 中定义了模型配置文件的全部信息：
![image.png](https://cdn.nlark.com/yuque/0/2023/png/22094377/1684921449647-bf314963-f123-44e8-aa45-a85bebad2faf.png#averageHue=%230f131a&clientId=u978888e7-226c-4&from=paste&height=570&id=ud6f7ce6c&originHeight=570&originWidth=648&originalType=binary&ratio=1&rotation=0&showTitle=false&size=33415&status=done&style=none&taskId=u1b188dce-75b6-4c46-af66-c97ef18206b&title=&width=648)
可以利用 protoc 工具来将 .proto 后缀的文件转换为一些常用的编程语言的文件，如 .py 文件，再借助这些转化来的 .py 文件就可以读取特定格式的 .pbtxt 文件。
#### 最小模型配置
最小模型配置需指明：platform、max_batch_size 以及模型的输入输出张量，如下所示：
```cpp
  platform: "tensorrt_plan"
  max_batch_size: 8
  input [
    {
      name: "input0"
      data_type: TYPE_FP32
      dims: [ 16 ]
    },
    {
      name: "input1"
      data_type: TYPE_FP32
      dims: [ 16 ]
    }
  ]
  output [
    {
      name: "output0"
      data_type: TYPE_FP32
      dims: [ 16 ]
    }
  ]
```

- max_batch_size：模型支持的可被Triton利用的批处理类型的最大批大小。如果模型的批处理维度是第一个维度，并且模型的所有输入和输出都具有该批处理维度，那么Triton可以使用其动态批处理程序或序列批处理程序自动与模型一起使用批处理。
- input / output tensor：每个模型输入和输出必须指定名称、数据类型和形状。为输入或输出张量指定的名称必须与模型期望的名称匹配。
#### 自动生成模型配置
Triton可以自动导出大多数 TensorRT、TensorFlow、ONNX 和 OpenVINO 模型所需的最小模型配置。同时，也可以使用 curl 和 [model configuration endpoint](https://docs.nvidia.com/deeplearning/triton-inference-server/user-guide/docs/protocol/extension_model_configuration.html) 对配置文件进行获取，如下所示：
```cpp
curl localhost:8000/v2/models/<model name>/config
```
#### Reshape
ModelTensorReshape 属性用于指示推理 API 接受的输入或输出形状不同于底层框架模型或自定义后端期望或产生的输入或输出形状，并对这种差异进行重塑。
```cpp
  input [
    {
      name: "in"
      dims: [ 1 ]
      reshape: { shape: [ ] }
    }
```
#### Version Policy
每个模型可以有一个或多个版本。模型配置的 ModelVersionPolicy 属性用于设置以下策略之一。

- All ：模型存储库中可用的模型的所有版本都可用于推理。Version_policy: {all: {}}
- Latest：只有存储库中模型的最新“n”版本可用于推断。该模型的最新版本是数字上最大的版本号。Version_policy: {latest: {num_versions: 2}}
- Specific：只有特定列出的模型版本可用于推断。version_policy: { specific: { versions: [1,3]}}
```cpp
  platform: "tensorrt_plan"
  max_batch_size: 8
  input [
    {
      name: "input0"
      data_type: TYPE_FP32
      dims: [ 16 ]
    },
    {
      name: "input1"
      data_type: TYPE_FP32
      dims: [ 16 ]
    }
  ]
  output [
    {
      name: "output0"
      data_type: TYPE_FP32
      dims: [ 16 ]
    }
  ]
  version_policy: { all { }}
```
在加载模型阶段，会将满足 version policy 的所有版本的模型加载进内存并提供服务；在请求服务阶段，通过指定模型的版本来获取对应版本的模型的推理服务。
```cpp
if (model_config.version_policy().has_specific()) {
    for (const auto& v : model_config.version_policy().specific().versions()) {
        // Only load the specific versions that are presented in model directory
        bool version_not_exist = existing_versions.insert(v).second;
        if (!version_not_exist) {
            versions->emplace(v);
        } else {
            LOG_ERROR << "version " << v << " is specified for model '" << model_id
                << "', but the version directory is not present";
        }
    }
} else {
    if (model_config.version_policy().has_latest()) {
        // std::set is sorted with std::greater
        for (const auto& v : existing_versions) {
            if (versions->size() >=
                model_config.version_policy().latest().num_versions()) {
                break;
            }
            versions->emplace(v);
        }
    } else {
        // all
        versions->insert(existing_versions.begin(), existing_versions.end());
    }
}
```
![image.png](https://cdn.nlark.com/yuque/0/2023/png/22094377/1684928389775-89fe402e-9098-4ccb-9bdc-23f6f51bacf0.png#averageHue=%23242322&clientId=u978888e7-226c-4&from=paste&height=104&id=uf880f30e&originHeight=104&originWidth=643&originalType=binary&ratio=1&rotation=0&showTitle=false&size=15061&status=done&style=none&taskId=u92df10b8-9c93-456a-ae46-160211fb05c&title=&width=643)
#### Model Warmup
当一个模型被Triton加载时，相应的后端需要为该模型进行初始化。对于某些后端，它们的初始化会被延迟（如计算图采用 Lazy Initialization 方式，导致第一次请求需要等待计算图初始化，[美团技术团队有一篇文章](https://tech.meituan.com/2018/10/11/tfserving-improve.html) 就提到了这个“模型切换毛刺问题”），直到模型接收到第一个推理请求。因此，由于延迟了后端的初始化，第一个推理请求可能会明显变慢。
为了避免这些初始的、缓慢的推理请求，Triton提供了一个配置选项 ModelWarmup，使模型能够“预热”，以便在接收到第一个推理请求之前完全初始化。在模型配置中定义 ModelWarmup 属性时，在模型预热完成之前，Triton不会将模型显示为已准备好进行推理。
![image.png](https://cdn.nlark.com/yuque/0/2023/png/22094377/1684918748532-e3184b59-b6ad-4db0-bc41-ec442477c121.png#averageHue=%23f6f6f6&clientId=u978888e7-226c-4&from=paste&height=321&id=u8531bfa7&originHeight=321&originWidth=357&originalType=binary&ratio=1&rotation=0&showTitle=false&size=49267&status=done&style=none&taskId=udd809d60-e43c-419c-a64a-3a833f9094d&title=&width=357)
# Triton Server 实现与扩展
## 函数调用逻辑
### 总览
![image.png](https://cdn.nlark.com/yuque/0/2023/png/22094377/1684220227779-17b8aab6-eb82-4e0b-9017-b773a64d309a.png#averageHue=%23b6ad56&clientId=uee690e7f-bb46-4&from=paste&height=575&id=u79db028d&originHeight=575&originWidth=990&originalType=binary&ratio=1&rotation=0&showTitle=false&size=38073&status=done&style=none&taskId=u44371460-2399-4698-8797-666b69a8b1e&title=&width=990)

![image.png](https://cdn.nlark.com/yuque/0/2023/png/22094377/1683982907122-ccadf676-f258-4c86-9557-c7cea90cd696.png#averageHue=%23efe5e1&clientId=u54f0b810-ac90-4&from=paste&height=574&id=pBaOQ&originHeight=574&originWidth=1850&originalType=binary&ratio=1&rotation=0&showTitle=false&size=385442&status=done&style=none&taskId=u6544f967-27ba-4de8-b7de-fc2dcf3dbd4&title=&width=1850)
Triton 从初始化 Server 到将模型加载进指定后端推理引擎的时序图如上所示：

- 首先初始化 InferenceServer，这个类就是整个 Triton 推理的入口；
- InferenceServer 在 Init() 方法中核心就是创建 ModelRepositoryManger 类，这个 ModelRepositoryManger 是模型仓库的管理类，它会到模型仓库地址下获取所有的模型；
- 对于每一个模型都会通过 ModelRepositoryManger::BackendLifeCycle::CreateInferenceBackend() 调用TritonBackendManager::CreateBackend()
### 模型拉取
![image.png](https://cdn.nlark.com/yuque/0/2023/png/22094377/1685339051072-1652f843-14b7-4372-b8d2-aa443dd94e86.png#averageHue=%23f9f9f9&clientId=u07a277f0-d48a-4&from=paste&height=919&id=u71ad48c3&originHeight=919&originWidth=1764&originalType=binary&ratio=1&rotation=0&showTitle=false&size=108141&status=done&style=none&taskId=u5f7d6b5d-ba3d-4f2a-8671-92ccc4070a4&title=&width=1764)
```cpp
Status
LocalizePath(const std::string& path, std::shared_ptr<LocalizedPath>* localized)
{
  std::shared_ptr<FileSystem> fs;
  RETURN_IF_ERROR(fsm_.GetFileSystem(path, fs));
  return fs->LocalizePath(path, localized);
}
```
由上图可知，Triton对于指定的 --model_repository 的解析最终会通过调用 filesystem.h 中暴露出来的 API 来实现。在 filesystem.cc 文件中定义了四种文件系统：
```
enum class FileSystemType { LOCAL, GCS, S3, AS };
```
其中每种文件系统都有其独立的命名空间，在其命名空间中各自实现了 filesystem.h 中暴露出来的所有方法，具体的实现细节则是利用各自云厂商的第三方库来新建一个 Client 。Triton 通过下面的函数来实现 Repo Path 的解析并选择合适的命名空间中的方法：
```cpp
Status
FileSystemManager::GetFileSystem(
const std::string& path, std::shared_ptr<FileSystem>& file_system)
{
    // Check if this is a GCS path (gs://$BUCKET_NAME)
    if (!path.empty() && !path.rfind("gs://", 0)) {
        #ifndef TRITON_ENABLE_GCS
        return Status(
            Status::Code::INTERNAL,
            "gs:// file-system not supported. To enable, build with "
            "-DTRITON_ENABLE_GCS=ON.");
        #else
        return GetFileSystem<
            std::vector<std::tuple<
            std::string, GCSCredential, std::shared_ptr<GCSFileSystem>>>,
            GCSCredential, GCSFileSystem>(path, gs_cache_, file_system);
        #endif  // TRITON_ENABLE_GCS
    }

    // Check if this is an S3 path (s3://$BUCKET_NAME)
    if (!path.empty() && !path.rfind("s3://", 0)) {
        #ifndef TRITON_ENABLE_S3
        return Status(
            Status::Code::INTERNAL,
            "s3:// file-system not supported. To enable, build with "
            "-DTRITON_ENABLE_S3=ON.");
        #else
        return GetFileSystem<
            std::vector<std::tuple<
            std::string, S3Credential, std::shared_ptr<S3FileSystem>>>,
            S3Credential, S3FileSystem>(path, s3_cache_, file_system);
        #endif  // TRITON_ENABLE_S3
    }

    // Check if this is an Azure Storage path
    if (!path.empty() && !path.rfind("as://", 0)) {
        #ifndef TRITON_ENABLE_AZURE_STORAGE
        return Status(
            Status::Code::INTERNAL,
            "as:// file-system not supported. To enable, build with "
            "-DTRITON_ENABLE_AZURE_STORAGE=ON.");
        #else
        return GetFileSystem<
            std::vector<std::tuple<
            std::string, ASCredential, std::shared_ptr<ASFileSystem>>>,
            ASCredential, ASFileSystem>(path, as_cache_, file_system);
        #endif  // TRITON_ENABLE_AZURE_STORAGE
    }

    // Assume path is for local filesystem
    file_system = local_fs_;
    return Status::Success;
}

template <class CacheType, class CredentialType, class FileSystemType>
Status
FileSystemManager::GetFileSystem(
    const std::string& path, CacheType& cache,
    std::shared_ptr<FileSystem>& file_system)
{
  const Status& cred_status = LoadCredentials();
  if (cred_status.IsOk() ||
      cred_status.StatusCode() == Status::Code::ALREADY_EXISTS) {
    // Find credential
    size_t idx;
    const Status& match_status = GetLongestMatchingNameIndex(cache, path, idx);
    if (!match_status.IsOk()) {
      return ReturnErrorOrReload<CacheType, CredentialType, FileSystemType>(
          cred_status, match_status, path, cache, file_system);
    }
    // Find or lazy load file system
    std::shared_ptr<FileSystemType> fs = std::get<2>(cache[idx]);
    if (fs == nullptr) {
      std::string cred_name = std::get<0>(cache[idx]);
      CredentialType cred = std::get<1>(cache[idx]);
      fs = std::make_shared<FileSystemType>(path, cred);
      cache[idx] = std::make_tuple(cred_name, cred, fs);
    }
    // Check client
    const Status& client_status = fs->CheckClient(path);
    if (!client_status.IsOk()) {
      return ReturnErrorOrReload<CacheType, CredentialType, FileSystemType>(
          cred_status, client_status, path, cache, file_system);
    }
    // Return client
    file_system = fs;
    return Status::Success;
  }
  return cred_status;
}


```
云存储的配置文件：
```cpp
export TRITON_CLOUD_CREDENTIAL_PATH="cloud_credential.json"

“cloud_credential.json”:
{
  "gs": {
    "": "PATH_TO_GOOGLE_APPLICATION_CREDENTIALS",
    "gs://gcs-bucket-002": "PATH_TO_GOOGLE_APPLICATION_CREDENTIALS_2"
  },
  "s3": {
    "": {
      "secret_key": "AWS_SECRET_ACCESS_KEY",
      "key_id": "AWS_ACCESS_KEY_ID",
      "region": "AWS_DEFAULT_REGION",
      "session_token": "",
      "profile": ""
    },
    "s3://s3-bucket-002": {
      "secret_key": "AWS_SECRET_ACCESS_KEY_2",
      "key_id": "AWS_ACCESS_KEY_ID_2",
      "region": "AWS_DEFAULT_REGION_2",
      "session_token": "AWS_SESSION_TOKEN_2",
      "profile": "AWS_PROFILE_2"
    }
  },
  "as": {
    "": {
      "account_str": "AZURE_STORAGE_ACCOUNT",
      "account_key": "AZURE_STORAGE_KEY"
    },
    "as://Account-002/Container": {
      "account_str": "",
      "account_key": ""
    }
  }
}
```
确定了仓库路径所属的文件系统后，Triton 会通过在 filesystem.cc 中实现的如下步骤来完成对模型仓库的拉取，下面以 S3 为例：
![image.png](https://cdn.nlark.com/yuque/0/2023/png/22094377/1684220252117-7647a547-7515-4c16-84f1-8e06fa41b7e4.png#averageHue=%236eaa46&clientId=uee690e7f-bb46-4&from=paste&height=722&id=u332363d6&originHeight=722&originWidth=1241&originalType=binary&ratio=1&rotation=0&showTitle=false&size=95380&status=done&style=none&taskId=uc1536687-3689-4167-9dd1-24d5a14de2d&title=&width=1241)
第一步，通过 LoadCredentials 加载证书文件（针对云端文件系统），将密钥等配置信息加载到 S3Credential 中。
```cpp
Status
FileSystemManager::LoadCredentials(bool flush_cache)
{
  // prevent concurrent access into class variables
  std::lock_guard<std::mutex> lock(mu_);

  // check if credential is already cached
  if (is_cached_ && !flush_cache) {
    return Status(Status::Code::ALREADY_EXISTS, "Cached");
  }

  const char* file_path_c_str = std::getenv("TRITON_CLOUD_CREDENTIAL_PATH");
  if (file_path_c_str != nullptr) {
    // Load from credential file
    std::string file_path = std::string(file_path_c_str);
    LOG_VERBOSE(1) << "Reading cloud credential from " << file_path;

    triton::common::TritonJson::Value creds_json;
    std::string cred_file_content;
    RETURN_IF_ERROR(local_fs_->ReadTextFile(file_path, &cred_file_content));
    RETURN_IF_ERROR(creds_json.Parse(cred_file_content));
    ···
    #ifdef TRITON_ENABLE_S3
        // load S3 credentials
        LoadCredential<
            std::vector<std::tuple<
                std::string, S3Credential, std::shared_ptr<S3FileSystem>>>,
            S3Credential, S3FileSystem>(creds_json, "s3", s3_cache_);
    #endif  // TRITON_ENABLE_S3
    ···
    #ifdef TRITON_ENABLE_S3
        // load S3 credentials
        s3_cache_.clear();
        s3_cache_.push_back(
            std::make_tuple("", S3Credential(), std::shared_ptr<S3FileSystem>()));
    #endif  // TRITON_ENABLE_S3
    ···
}

template <class CacheType, class CredentialType, class FileSystemType>
void
FileSystemManager::LoadCredential(
    triton::common::TritonJson::Value& creds_json, const char* fs_type,
    CacheType& cache)
{
  cache.clear();
  triton::common::TritonJson::Value creds_fs_json;
  if (creds_json.Find(fs_type, &creds_fs_json)) {
    std::vector<std::string> cred_names;
    creds_fs_json.Members(&cred_names);
    for (size_t i = 0; i < cred_names.size(); i++) {
      std::string cred_name = cred_names[i];
      triton::common::TritonJson::Value cred_json;
      creds_fs_json.Find(cred_name.c_str(), &cred_json);
      cache.push_back(std::make_tuple(
          cred_name, CredentialType(cred_json),
          std::shared_ptr<FileSystemType>()));
    }
    SortCache(cache);
  }
}
```
```cpp
struct S3Credential {
  std::string secret_key_;
  std::string key_id_;
  std::string region_;
  std::string session_token_;
  std::string profile_name_;

  S3Credential();  // from env var
  S3Credential(triton::common::TritonJson::Value& cred_json);
};

S3Credential::S3Credential()
{
  const auto to_str = [](const char* s) -> std::string {
    return (s != nullptr ? std::string(s) : "");
  };
  const char* secret_key = std::getenv("AWS_SECRET_ACCESS_KEY");
  const char* key_id = std::getenv("AWS_ACCESS_KEY_ID");
  const char* region = std::getenv("AWS_DEFAULT_REGION");
  const char* session_token = std::getenv("AWS_SESSION_TOKEN");
  const char* profile = std::getenv("AWS_PROFILE");
  secret_key_ = to_str(secret_key);
  key_id_ = to_str(key_id);
  region_ = to_str(region);
  session_token_ = to_str(session_token);
  profile_name_ = to_str(profile);
}

S3Credential::S3Credential(triton::common::TritonJson::Value& cred_json)
{
  triton::common::TritonJson::Value secret_key_json, key_id_json, region_json,
      session_token_json, profile_json;
  if (cred_json.Find("secret_key", &secret_key_json))
    secret_key_json.AsString(&secret_key_);
  if (cred_json.Find("key_id", &key_id_json))
    key_id_json.AsString(&key_id_);
  if (cred_json.Find("region", &region_json))
    region_json.AsString(&region_);
  if (cred_json.Find("session_token", &session_token_json))
    session_token_json.AsString(&session_token_);
  if (cred_json.Find("profile", &profile_json))
    profile_json.AsString(&profile_name_);
}
```
第二步，新建 client_ 客户端，后续对云存储的一系列操作基于这个客户端。
```cpp
S3FileSystem::S3FileSystem(
    const std::string& s3_path, const S3Credential& s3_cred)
    : s3_regex_(
          "s3://(http://|https://|)([0-9a-zA-Z\\-.]+):([0-9]+)/"
          "([0-9a-z.\\-]+)(((/[0-9a-zA-Z.\\-_]+)*)?)")
{
  // init aws api if not already
  Aws::SDKOptions options;
  static std::once_flag onceFlag;
  std::call_once(onceFlag, [&options] { Aws::InitAPI(options); });

  Aws::Client::ClientConfiguration config;
  Aws::Auth::AWSCredentials credentials;

  // check vars for S3 credentials -> aws profile -> default
  if (!s3_cred.secret_key_.empty() && !s3_cred.key_id_.empty()) {
    credentials.SetAWSAccessKeyId(s3_cred.key_id_.c_str());
    credentials.SetAWSSecretKey(s3_cred.secret_key_.c_str());
    if (!s3_cred.session_token_.empty()) {
      credentials.SetSessionToken(s3_cred.session_token_.c_str());
    }
    config = Aws::Client::ClientConfiguration();
    if (!s3_cred.region_.empty()) {
      config.region = s3_cred.region_.c_str();
    }
  } else if (!s3_cred.profile_name_.empty()) {
    config = Aws::Client::ClientConfiguration(s3_cred.profile_name_.c_str());
  } else {
    config = Aws::Client::ClientConfiguration("default");
  }

  // Cleanup extra slashes
  std::string clean_path;
  LOG_STATUS_ERROR(CleanPath(s3_path, &clean_path), "failed to parse S3 path");

  std::string protocol, host_name, host_port, bucket, object;
  if (RE2::FullMatch(
          clean_path, s3_regex_, &protocol, &host_name, &host_port, &bucket,
          &object)) {
    config.endpointOverride = Aws::String(host_name + ":" + host_port);
    if (protocol == "https://") {
      config.scheme = Aws::Http::Scheme::HTTPS;
    } else {
      config.scheme = Aws::Http::Scheme::HTTP;
    }
  }

  if (!s3_cred.secret_key_.empty() && !s3_cred.key_id_.empty()) {
    client_ = std::make_unique<s3::S3Client>(
        credentials, config,
        Aws::Client::AWSAuthV4Signer::PayloadSigningPolicy::Never,
        /*useVirtualAdressing*/ false);
  } else {
    client_ = std::make_unique<s3::S3Client>(
        config, Aws::Client::AWSAuthV4Signer::PayloadSigningPolicy::Never,
        /*useVirtualAdressing*/ false);
  }
}
```
第三步，使用 ParsePath 函数解析 s3_path 参数（格式为 s3://bucket_name/object_path），将其分割为桶名 bucket 和对象路径 object_path。然后程序通过 S3 客户端的 HeadBucketRequest 函数检查能否连接到指定的存储桶。
```cpp
#ifdef TRITON_ENABLE_S3
#include <aws/core/Aws.h>
#include <aws/core/auth/AWSCredentialsProvider.h>
#include <aws/s3/S3Client.h>
#include <aws/s3/model/GetObjectRequest.h>
#include <aws/s3/model/HeadBucketRequest.h>
#include <aws/s3/model/HeadObjectRequest.h>
#include <aws/s3/model/ListObjectsRequest.h>
#endif  // TRITON_ENABLE_S3

Status
S3FileSystem::CheckClient(const std::string& s3_path)
{
  std::string bucket, object_path;
  RETURN_IF_ERROR(ParsePath(s3_path, &bucket, &object_path));
  // check if can connect to the bucket
  s3::Model::HeadBucketRequest head_request;
  head_request.WithBucket(bucket.c_str());
  if (!client_->HeadBucket(head_request).IsSuccess()) {
    return Status(
        Status::Code::INTERNAL,
        "Unable to create S3 filesystem client. Check account credentials.");
  }
  return Status::Success;
```
第四步，通过 FileExists 、IsDirectory 、GetDirectoryContents 、GetDirectorySubdirs 等函数进行文件的必要性检查并把需要下载的内容放入 contents 容器中。
第四步，通过 LocalizePath 函数下载模型仓库到本地暂存目录。
```cpp
Status
S3FileSystem::LocalizePath(
    const std::string& path, std::shared_ptr<LocalizedPath>* localized)
{
    ···
        s3::Model::GetObjectRequest object_request;
        object_request.SetBucket(file_bucket.c_str());
        object_request.SetKey(file_object.c_str());

        auto get_object_outcome = client_->GetObject(object_request);
        if (get_object_outcome.IsSuccess()) {
          auto& retrieved_file =
              get_object_outcome.GetResultWithOwnership().GetBody();
          std::ofstream output_file(local_fpath.c_str(), std::ios::binary);
          output_file << retrieved_file.rdbuf();
          output_file.close();
        } else {
          return Status(
              Status::Code::INTERNAL,
              "Failed to get object at " + s3_fpath + " due to exception: " +
                  get_object_outcome.GetError().GetExceptionName() +
                  ", error message: " +
                  get_object_outcome.GetError().GetMessage());
        }
    ···
}
```
第五步，Triton 从本地暂存目录 load 模型。
### 模型加载
![image.png](https://cdn.nlark.com/yuque/0/2023/png/22094377/1684067896284-e8df357f-d449-42f9-a267-eafe5093fa75.png#averageHue=%23f3f3f3&clientId=u8b03e6f8-53a6-4&from=paste&height=346&id=mbfJq&originHeight=346&originWidth=731&originalType=binary&ratio=1&rotation=0&showTitle=false&size=29256&status=done&style=none&taskId=u18351cc7-6c08-4730-a3a6-043fc1782ec&title=&width=731)
### HTTP 服务建立
![image.png](https://cdn.nlark.com/yuque/0/2023/png/22094377/1684063967622-2b0186c0-ffa8-4d9d-9e09-8987c8e329d4.png#averageHue=%23f8f8f8&clientId=uec66f8ec-7d12-4&from=paste&height=534&id=u3bcb074a&originHeight=534&originWidth=1368&originalType=binary&ratio=1&rotation=0&showTitle=false&size=51556&status=done&style=none&taskId=u7a2d0b63-e40e-424d-957c-6c044aa664c&title=&width=1368)

## API
### Model Repository
```
## Index 返回模型仓库中所有可用的模型的信息，包括那些暂时还没被加载进内存的模型。
POST v2/repository/index
$repository_index_request =
{
  "ready" : $boolean #optional,
}

$repository_index_response =
[
  {
    "name" : $string,
    "version" : $string #optional,
    "state" : $string,
    "reason" : $string
  },
  …
]

$repository_index_error_response =
{
  "error": $string
}

## Load 加载或者再加载模型进内存。
POST v2/repository/models/${MODEL_NAME}/load
$repository_load_request =
{
  "parameters" : $parameters #optional
}

$repository_load_error_response =
{
  "error": $string
}

## Unload 从Triton中卸载一个模型。
POST v2/repository/models/${MODEL_NAME}/unload
$repository_unload_request =
{
  "parameters" : $parameters #optional
}

$repository_unload_error_response =
{
  "error": $string
}
```
```
## Index
message RepositoryIndexRequest
{
  // The name of the repository. If empty the index is returned
  // for all repositories.
  string repository_name = 1;

  // If true return only models currently ready for inferencing.
  bool ready = 2;
}

message RepositoryIndexResponse
{
  // Index entry for a model.
  message ModelIndex {
    // The name of the model.
    string name = 1;

    // The version of the model.
    string version = 2;

    // The state of the model.
    string state = 3;

    // The reason, if any, that the model is in the given state.
    string reason = 4;
  }

  // An index entry for each model.
  repeated ModelIndex models = 1;
}

## Load
message RepositoryModelLoadRequest
{
  // The name of the repository to load from. If empty the model
  // is loaded from any repository.
  string repository_name = 1;

  // The name of the model to load, or reload.
  string model_name = 2;

  // Optional parameters.
  map<string, ModelRepositoryParameter> parameters = 3;
}

message RepositoryModelLoadResponse
{
}

## Unload
message RepositoryModelUnloadRequest
{
  // The name of the repository from which the model was originally
  // loaded. If empty the repository is not considered.
  string repository_name = 1;

  // The name of the model to unload.
  string model_name = 2;

  // Optional parameters.
  map<string, ModelRepositoryParameter> parameters = 3;
}

message RepositoryModelUnloadResponse
{
}
```
### Model Configuration
```bash
# 获取模型配置信息
GET v2/models/${MODEL_NAME}[/versions/${MODEL_VERSION}]/config

$model_configuration_response =
{
# configuration JSON
}

$model_configuration_error_response =
{
"error": <error message string>
}
```
```bash
# 获取模型配置信息
service GRPCInferenceService
{
  …

  // Get model configuration.
  rpc ModelConfig(ModelConfigRequest) returns (ModelConfigResponse) {}
}

message ModelConfigRequest
{
  // The name of the model.
  string name = 1;

  // The version of the model. If not given the version of the model
  // is selected automatically based on the version policy.
  string version = 2;
}

message ModelConfigResponse
{
  // The model configuration.
  ModelConfig config = 1;
}
```
### 推理请求
```
POST /v2/models/mymodel/infer HTTP/1.1
Host: localhost:8000
Content-Type: application/json
Content-Length: <xx>
{
  "id" : "42",
  "inputs" : [
    {
      "name" : "input0",
      "shape" : [ 2, 2 ],
      "datatype" : "UINT32",
      "data" : [ 1, 2, 3, 4 ]
    }
  ],
  "outputs" : [
    {
      "name" : "output0",
      "parameters" : { "classification" : 2 }
    }
  ]
}

HTTP/1.1 200 OK
Content-Type: application/json
Content-Length: <yy>
{
  "id" : "42"
  "outputs" : [
    {
      "name" : "output0",
      "shape" : [ 2 ],
      "datatype"  : "STRING",
      "data" : [ "3.3:1", "2.4:3" ]
    }
  ]
}
```
```
ModelInferRequest {
  model_name : "mymodel"
  model_version : -1
  inputs [
    {
      name : "input0"
      shape : [ 2, 2 ]
      datatype : "UINT32"
      contents { int_contents : [ 1, 2, 3, 4 ] }
    }
  ]
  outputs [
    {
      name : "output0"
      parameters [
        {
          key : "classification"
          value : { int64_param : 2 }
        }
      ]
    }
  ]
}

ModelInferResponse {
  model_name : "mymodel"
  outputs [
    {
      name : "output0"
      shape : [ 2 ]
      datatype  : "STRING"
      contents { bytes_contents : [ "3.3:1", "2.4:3" ] }
    }
  ]
}
```
### 模型状态
```bash
GET v2/models[/${MODEL_NAME}[/versions/${MODEL_VERSION}]]/stats

$stats_model_response =
{
"model_stats" : [ $model_stat, ... ]
}

$model_stat =
{
"name" : $string,
"version" : $string #optional,
"last_inference" : $number,
"inference_count" : $number, # 接收推理请求的次数
"execution_count" : $number, # 执行推理成功的次数
"inference_stats" : $inference_stats, # 推理成功或失败的次数
"batch_stats" : [ $batch_stat, ... ] # 在模型中执行的每个不同批大小的汇总统计信息
}

$repository_statistics_error_response =
{
"error": $string
}
```
```bash
service GRPCInferenceService
{
  …

  // Get the cumulative statistics for a model and version.
  rpc ModelStatistics(ModelStatisticsRequest)
          returns (ModelStatisticsResponse) {}
}

message ModelStatisticsRequest
{
  // The name of the model. If not given returns statistics for all
  // models.
  string name = 1;

  // The version of the model. If not given returns statistics for
  // all model versions.
  string version = 2;
}

message ModelStatisticsResponse
{
  // Statistics for each requested model.
  repeated ModelStatistics model_stats = 1;
}
```
## Repository Agent
![image.png](https://cdn.nlark.com/yuque/0/2023/png/22094377/1685368768234-38ce671c-bd2c-4cd9-8df4-80605dc8bbce.png#averageHue=%23141b22&clientId=u3558ce23-0aad-4&from=paste&height=586&id=u74c90dac&originHeight=1172&originWidth=1848&originalType=binary&ratio=2&rotation=0&showTitle=false&size=537734&status=done&style=none&taskId=ud7e65185-c1ef-443c-9055-b6d17284d4d&title=&width=924)
Repo Agent 可以在模型 Load 或者 Unload 时扩展 Triton 的功能。Repo Agent 与 Triton 通过 [repository agent API](https://github.com/triton-inference-server/core/tree/main/include/triton/core/tritonrepoagent.h) 来进行通信。
### 使用
通过在 [model configuration](https://github.com/triton-inference-server/server/blob/main/docs/user_guide/model_configuration.md) 的 ModelRepositoryAgents 字段中进行声明，一个模型可以使用一个或者多个 Repo Agent。每个 Agent 可以有自己特定的参数来控制它们的行为，如下，在一个模型被 Load 或者 Unload 时，这个模型指定的 Agent 会被按顺序依次唤醒并执行相应的功能。
```cpp
model_repository_agents
{
  agents [
    {
      name: "agent0",
      parameters [
        {
          key: "key0",
          value: "value0"
        },
        {
          key: "key1",
          value: "value1"
        }
      ]
    },
    {
      name: "agent1",
      parameters [
        {
          key: "keyx",
          value: "valuex"
        }
      ]
    }
  ]
}
```
### 实现
Repo Agent 必须作为共享库实现，并且共享库的名称必须为 libtritonrepoagent_<repo-agent-name>.so。共享库需要隐藏掉所有的 Symbols 除了那些被 [repository agent API](https://github.com/triton-inference-server/core/tree/main/include/triton/core/tritonrepoagent.h) 所需要的。
共享库会在需要的时候被Triton动态地加载。对于一个名称为 A 的 Repo Agent ，共享库必须位于<repository_agent_directory>/A/libtritonrepoagent_A.so，其中 <repository_agent_directory> 默认为 /opt/tritonserver/repoagents，可以通过 --repoagent-directory 参数来进行修改。
Triton通过以下步骤来 Load 一个模型：

- 加载模型的配置文件 config.pbtxt ，提取出配置文件中 ModelRepositoryAgents 部分的设置。后续整个加载流程中将采用初始时解析的配置文件，即使 Agent 对配置文件作出修改。
- 根据配置文件中的设置初始化对应的 repository agent，加载相应的共享库。共享库初始化失败会导致模型加载失败。
- 通过 TRITONREPOAGENT_ACTION_LOAD 动作唤醒 repositoy agent 的 TRITONREPOAGENT_ModelAction 函数，该函数位于共享库中，包含了用户自定义操作的实现。
- 代理将返回 success 或者 failure 来表明对于模型仓库的更改状态，并且可以创建一个新的仓库地址来存放修改后的模型。
- 如果所有的代理返回 success ，Triton 会使用最后一个模型仓库的位置来加载模型。
### 函数调用逻辑
![image.png](https://cdn.nlark.com/yuque/0/2023/png/22094377/1684475642501-0b275c83-f0c4-4df8-9a30-939db7eceb4d.png#averageHue=%23f8f8f8&clientId=u61a3487a-13f8-4&from=paste&height=731&id=uee4a7113&originHeight=731&originWidth=2519&originalType=binary&ratio=1&rotation=0&showTitle=false&size=120068&status=done&style=none&taskId=ue756a251-3900-4a2d-b3f9-c58c5a22ac5&title=&width=2519)
上图为使用了 Repo Agent 后 Triton加载模型的函数调用逻辑。
其中，Poll 函数会首先通过 GetDirectorySubdirs 函数对指定的模型仓库地址进行有效性校验，同时获取仓库目录下面所有的模型目录。继而对于每一个模型通过 InitializeModelInfo 函数初始化模型的一些信息，这是在模型加载进 Triton 之前关键的一步。首先，该函数会获取模型的修改时间来判断需不需要进行再加载（Poll 模式），然后通过 ReadTextProto 函数读取模型配置文件的信息并解析出其中对于 ModelRepositoryAgents 部分的设置，并利用 CreateAgentModelListWithLoadAciton 函数初始化对应的 repository agent。
具体的，通过下面这个函数来判断最新的模型是否被加载（模型是否发生修改）：
```cpp
bool
IsModified(
    const std::string& model_dir_path, std::pair<int64_t, int64_t>* last_ns)
{
  auto new_ns = GetDetailedModifiedTime(model_dir_path);
  bool modified = std::max(new_ns.first, new_ns.second) >
                  std::max(last_ns->first, last_ns->second);
  last_ns->swap(new_ns);
  return modified;
}
```
CreateAgentModelListWithLoadAciton 函数会去到指定的目录下（默认为 /opt/tritonserver/repoagents/）找到配置文件中指定的代理，利用 TritonRepoAgent::Create() 函数将共享库暴露出来的 TRITONREPOAGENT_ModelAction 作为一个回调函数 model_action_fn_。然后通过 TritonRepoAgentModel::Create() 函数创建一个 AgentModel，它留存了这个模型对应的地址，后续模型的加载等操作将基于此地址，最后，InvokeAgent 函数将调用前面设置的回调函数触发共享库内部实现的操作，如模型的解密和模型地址的修改等。
## 扩展实现
为了实现本项目的最终目的，即将 Triton 对于模型仓库拉取的请求重定向到 Dragonfly，需要借助 Repository Agent 进行实现。

- 编写共享库，实现其中的 TRITONREPOAGENT_ModelAction 函数，函数的作用是将拉取请求和仓库地址转发给 Dragonfly。
- 将代码编译成 .so 文件，命名格式为 libtritonrepoagent_<repo-agent-name>.so ，并将该共享库置于 /opt/tritonserver/<repo-agent-name>/ 目录下。
- 在模型的配置文件 config.pbtxt 中使用编写好的 repository agent。

完成上述步骤后，当 Triton 请求拉取一个模型仓库的时候，会根据解析到的模型配置文件中的 repository agent 字段动态加载共享库，将请求和仓库地址转发到 Dragonfly，Dragonfly根据请求和地址进行拉取，拉取成功后返回给 Triton 新的模型的地址（本地文件系统地址），Triton 根据该地址加载模型。
### Demo
```cpp
// 需要实现的 API
TRITONSERVER_Error*
TRITONREPOAGENT_ModelAction(
    TRITONREPOAGENT_Agent* agent, TRITONREPOAGENT_AgentModel* model,
    const TRITONREPOAGENT_ActionType action_type)
{
  // Return success (nullptr) if the agent does not handle the action
  if (action_type != TRITONREPOAGENT_ACTION_LOAD) {
    return nullptr;
  }

  // 通过 ModelRepositoryLocation 传入模型初始地址 location
  const char* location_cstr;
  TRITONREPOAGENT_ArtifactType artifact_type;
  RETURN_IF_ERROR(TRITONREPOAGENT_ModelRepositoryLocation(
      agent, model, &artifact_type, &location_cstr));
  const std::string location(location_cstr);

  // 通过环境变量传入云存储证书文件信息
  const char* file_path_c_str = std::getenv("TRITON_CLOUD_CREDENTIAL_PATH");
  std::string cred_path;
  if (file_path_c_str != nullptr) {
    // Load from credential file
    cred_path = std::string(file_path_c_str);
  }

  // 通过 ModelParameterCount 传入 d7y 配置文件信息
  // TODO：用环境变量的方式传入是否更好？
  // Check the agent parameters for the demo configuration of the model
  uint32_t idx = 0;
  const char* key = nullptr;
  const char* value = nullptr;
  RETURN_IF_ERROR(
      TRITONREPOAGENT_ModelParameter(agent, model, idx, &key, &value));
  const auto d7yconf_path = value;

  try {
    const char* new_path = Dragonfly(d7yconf_path, cred_path, location);
    // 通过以下方式更新模型仓库地址
    RETURN_IF_ERROR(TRITONREPOAGENT_ModelRepositoryUpdate(
      agent, model, TRITONREPOAGENT_ARTIFACT_FILESYSTEM, new_path));
  }
  catch (const ErrorException& ee) {
    return ee.err_;
  }

  return nullptr;  // success
}

```
```cpp
name: "simple"
platform: "tensorflow_graphdef"
max_batch_size: 8
input [
  {
    name: "INPUT0"
    data_type: TYPE_INT32
    dims: [ 16 ]
  },
  {
    name: "INPUT1"
    data_type: TYPE_INT32
    dims: [ 16 ]
  }
]
output [
  {
    name: "OUTPUT0"
    data_type: TYPE_INT32
    dims: [ 16 ]
  },
  {
    name: "OUTPUT1"
    data_type: TYPE_INT32
    dims: [ 16 ]
  }
]
version_policy: { all { }}
model_repository_agents
{
  agents [
    {
      name: "demo",
      parameters
      {
        key: "config_path",
        value: "d7y_config.json"
      }
    }
  ]
}
```
![image.png](https://cdn.nlark.com/yuque/0/2023/png/22094377/1685348203858-5dae7078-eecd-42e7-8828-d9307aedb61e.png#averageHue=%23201f1e&clientId=uc19bfa2c-af62-4&from=paste&height=160&id=ubc3f40c2&originHeight=160&originWidth=460&originalType=binary&ratio=1&rotation=0&showTitle=false&size=7529&status=done&style=none&taskId=u1dab42e4-5dc9-4875-9b78-a05cc224b3b&title=&width=460)
![image.png](https://cdn.nlark.com/yuque/0/2023/png/22094377/1685348288536-23f22951-0452-49ae-81f0-f9df3be4c64e.png#averageHue=%232a2623&clientId=uc19bfa2c-af62-4&from=paste&height=90&id=u678a136a&originHeight=90&originWidth=497&originalType=binary&ratio=1&rotation=0&showTitle=false&size=7940&status=done&style=none&taskId=u2d676c65-6d55-4022-aace-318629f899e&title=&width=497)
不使用自定义的 demo repoagent 时，Triton 会使用自身实现的 S3FileSystem::LocalizePath() 函数将模型仓库下载到本地暂存文件：
![image.png](https://cdn.nlark.com/yuque/0/2023/png/22094377/1685348371600-8b9bd3f1-5721-46f3-b561-62247ee11cf1.png#averageHue=%23242220&clientId=uc19bfa2c-af62-4&from=paste&height=471&id=uae6f28e9&originHeight=471&originWidth=1613&originalType=binary&ratio=1&rotation=0&showTitle=false&size=101560&status=done&style=none&taskId=u722922c8-cfa6-4c3a-a700-e5791de6db0&title=&width=1613)
![image.png](https://cdn.nlark.com/yuque/0/2023/png/22094377/1685348398968-5424fcd3-885c-4c59-be72-9683c8920fb5.png#averageHue=%2323211f&clientId=uc19bfa2c-af62-4&from=paste&height=29&id=u1dccf41c&originHeight=29&originWidth=345&originalType=binary&ratio=1&rotation=0&showTitle=false&size=3323&status=done&style=none&taskId=ue4c01f7a-8cba-4e65-8249-1c6d0f24792&title=&width=345)
使用自定义的 demo repoagent 时，Triton 不会将模型仓库下载到本地暂存文件，而是更新模型仓库地址，从新的本地文件系统地址加载模型仓库：
![image.png](https://cdn.nlark.com/yuque/0/2023/png/22094377/1685348443001-64a966dd-9fbc-40d7-b56b-304bc5c17edc.png#averageHue=%23242220&clientId=uc19bfa2c-af62-4&from=paste&height=332&id=u0966a1b6&originHeight=332&originWidth=1616&originalType=binary&ratio=1&rotation=0&showTitle=false&size=67305&status=done&style=none&taskId=udbddcde7-d83a-4830-abc7-ada9462a80a&title=&width=1616)
![image.png](https://cdn.nlark.com/yuque/0/2023/png/22094377/1685348480934-84a2bcbf-289a-4604-b74a-733433e09a2e.png#averageHue=%23342d28&clientId=uc19bfa2c-af62-4&from=paste&height=42&id=u5dd920ba&originHeight=42&originWidth=216&originalType=binary&ratio=1&rotation=0&showTitle=false&size=1873&status=done&style=none&taskId=u89399f5b-3150-4df7-96c0-4488d4a1f64&title=&width=216)
### 不足

- 目前的 repository agent 仅支持以模型为单位进行操作，即在模型仓库的每个模型的配置文件中都需要配置对应字段。是否有支持对于整个仓库进行操作的方法？
