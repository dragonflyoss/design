# TorchServe

## Torch serve 总览
### Torch Serve 框架
Torch Serve 被设计成一个多模型推理框架。它可以通过 API 来请求推理，也可以通过 API 来管理模型，同时跟踪日志。Torch Serve 允许同时管理多个工作进程，这些工作进程动态会分配给不同的模型，工作进程的行为由自定义代码和存储的模型及权重确定。Torch Serve 分为前端和后端。前端使用 JAVA/C++ 代码，后端使用 Python 代码。
![image.png](https://cdn.nlark.com/yuque/0/2023/png/35887099/1683498952428-c295b695-42b6-42e4-873f-fe34ab03e497.png#averageHue=%23fcfdfa&clientId=u2e70bfff-66b3-4&from=paste&height=420&id=KQO0K&originHeight=840&originWidth=1374&originalType=binary&ratio=2&rotation=0&showTitle=false&size=173367&status=done&style=none&taskId=u496042f3-768e-418b-bf6b-43defe55806&title=&width=687)

- **Frontend**: Torch Serve 的 Request/Response 处理组件。组件的这一部分用来处理来自客户端的 Request/Response 并管理模型的生命周期。
- **API:** 支持 REST 和 gRPC API 格式：

REST: 默认在端口8080监听推理 API 正在端口8081监听管理 API。
gRPC: 默认在端口7070监听推理 API 正在端口7071监听管理 API。

- **Model Workers**: 并行处理模型推理请求，一个 Worker 加载一个模型副本。
- **Model**: 模型可以是 script_module (JIT saved models) 或 eager_mode_models 模式。 这些模型可以提供自定义的数据预处理和后处理以及任何其他工作模型。可以从云存储或本地主机加载模型。
- **Plugins**: 这些是自定义端点或 authz/authn 或批处理算法，可以在启动时放入 TorchServe。
- **Model Store**: 存储所有可加载模型的目录。

### Torch Serve 支持后端存储方式
Torch Serve 支持多种后端存储方式（模型优化框架），包括：Torchscript、ORT和ONNX、IPEX、TensorRT、FasterTransformer。可以在模型加载时通过 --serialized-file 加载不同格式的模型张量来使用不同框架。 Torch Serve 支持从本地文件系统或符合标准的 URL （用户可以在配置文件中指定 allowed_urls 参数，Torch Serve 将默认只接受来自 http://localhost 和 file:// 的 URL。）下载封装好的模型，并且支持 AWS S3 的服务器端加密。
```java
public static boolean downloadArchive(
            List<String> allowedUrls,
            File location,
            String archiveName,
            String url,
            boolean s3SseKmsEnabled)
            throws FileAlreadyExistsException, FileNotFoundException, DownloadArchiveException,
                    InvalidArchiveURLException {
        if (validateURL(allowedUrls, url)) {
            //本地
            if (location.exists()) {
                throw new FileAlreadyExistsException(archiveName);
            }
            try {
                // url
                HttpUtils.copyURLToFile(new URL(url), location, s3SseKmsEnabled);
            } catch (IOException e) {
                FileUtils.deleteQuietly(location);
                throw new DownloadArchiveException("Failed to download archive from: " + url, e);
            }
        }

        return true;
    }
}
```

| Torchscript | TorchScript 是 PyTorch 的子集，它可以通过捕获您的 Python 代码来创建可序列化和优化的模型。 | model.pt 
model.pth |
| --- | --- | --- |
| ONNX | ONNX 是一个开放的模型交换格式，它使模型可以在不同的深度学习框架之间进行转换和交换。 | model.onxx |
| IPEX | IPEX 是一个用于在 Intel 硬件上加速 PyTorch 的库。它提供了一种方法来优化和运行特定于 Intel 架构的 PyTorch 模型。 | model.pt 
model.pth |
| TensorRT | TensorRT 是 NVIDIA 的深度学习模型优化器和运行时库，用于在 NVIDIA GPU 上提供快速推理。 | model.onxx |
| FasterTransformer | FasterTransformer 是 NVIDIA 的库，它优化了 Transformer 模型（一种在许多 NLP 任务中使用的深度学习模型）在 NVIDIA GPU 上的运行。 | model.pt 
model.pth |


### Torch Serve 简单用例
```shell
# clone torch serve
git clone https://github.com/pytorch/serve.git

#creat model store and archiver model files
mkdir model_store
wget https://download.pytorch.org/models/densenet161-8d451a50.pth
torch-model-archiver --model-name densenet161 --version 1.0 --model-file ./serve/examples/image_classifier/densenet_161/model.py --serialized-file densenet161-8d451a50.pth --export-path model_store --extra-files ./serve/examples/image_classifier/index_to_name.json --handler image_classifier

#strat torch serve and load model densenet161.mar
torchserve --start --ncs --model-store model_store --models densenet161.mar

#predict model by REST inference API
curl http://127.0.0.1:8080/predictions/densenet161 -T kitten_small.jpg

#[Output]
[
{"tiger_cat": 0.46933549642562866},
{"tabby": 0.4633878469467163},
{"Egyptian_cat": 0.06456148624420166},
{"lynx": 0.0012828214094042778},
{"plastic_bag": 0.00023323034110944718}
]

#stop torch serve
torchserve --stop
```

## Torch Serve 的部署及模型生命周期
**Require:**
python > =3.8, JAVA>=11
git clone [https://github.com/pytorch/serve.git](https://github.com/pytorch/serve.git)
install dependencies
### Archiver model
Torch Serve 的一个关键特性是能够将模型工件打包到单个模型存档文件中。它是一个单独的命令行（CLI）torch-model-archiver。可以将模型检查点或模型定义文件打包到一个 .mar 文件中。任何使用 Torch Serve 的人都可以重新分发和提供该文件。在 eager_mode_models 下，它需要一个模型定义文件和stat_dict(.pt) 文件。而在 script_module 下只需要一个checkpoint(.pt) 文件。torch-model-archiver 只能在本地文件系统内打包，但是可以从云端下载封装好的模型文件。
```
$ torch-model-archiver -h
usage: torch-model-archiver [-h] --model-name MODEL_NAME  --version MODEL_VERSION_NUMBER
                      --model-file MODEL_FILE_PATH --serialized-file MODEL_SERIALIZED_PATH
                      --handler HANDLER [--runtime {python,python3}]
                      [--export-path EXPORT_PATH] [-f] [--requirements-file] [--config-file]

Model Archiver Tool

optional arguments:
  -h, --help            show this help message and exit
  --model-name MODEL_NAME
                        Exported model name. Exported file will be named as
                        model-name.mar and saved in current working directory
                        if no --export-path is specified, else it will be
                        saved under the export path
  --serialized-file SERIALIZED_FILE
                        Path to .pt or .pth file containing state_dict in
                        case of eager mode or an executable ScriptModule
                        in case of TorchScript.
  --model-file MODEL_FILE
                        Path to python file containing model architecture.
                        This parameter is mandatory for eager mode models.
                        The model architecture file must contain only one
                        class definition extended from torch.nn.Module.
  --handler HANDLER     TorchServe's default handler name  or handler python
                        file path to handle custom TorchServe inference logic.
  --extra-files EXTRA_FILES
                        Comma separated path to extra dependency files.
  --runtime {python,python3}
                        The runtime specifies which language to run your
                        inference code on. The default runtime is
                        RuntimeType.PYTHON. At the present moment we support
                        the following runtimes python, python3
  --export-path EXPORT_PATH
                        Path where the exported .mar file will be saved. This
                        is an optional parameter. If --export-path is not
                        specified, the file will be saved in the current
                        working directory.
  --archive-format {tgz, no-archive, zip-store, default}
                        The format in which the model artifacts are archived.
                        "tgz": This creates the model-archive in <model-name>.tar.gz format.
                        If platform hosting requires model-artifacts to be in ".tar.gz"
                        use this option.
                        "no-archive": This option creates an non-archived version of model artifacts
                        at "export-path/{model-name}" location. As a result of this choice,
                        MANIFEST file will be created at "export-path/{model-name}" location
                        without archiving these model files
                        "zip-store": This creates the model-archive in <model-name>.mar format
                        but will skip deflating the files to speed up creation. Mainly used
                        for testing purposes
                        "default": This creates the model-archive in <model-name>.mar format.
                        This is the default archiving format. Models archived in this format
                        will be readily hostable on TorchServe.
  -f, --force           When the -f or --force flag is specified, an existing
                        .mar file with same name as that provided in --model-
                        name in the path specified by --export-path will
                        overwritten
  -v, --version         Model's version.
  -r, --requirements-file
                        Path to requirements.txt file containing a list of model specific python
                        packages to be installed by TorchServe for seamless model serving.
  -c, --config-file         Path to a model config yaml file.
```

这一部分的功能主要通过[model_archiver](https://github.com/pytorch/serve/tree/master/model-archiver/model_archiver)下的 Python 代码实现，先通过 manifest_components 文件夹下的 Python 文件解析 CLI 生成 MANIFEST.json 文件，再利用 model_packaging.py 将所有模型相关文件进行压缩，放入 model_store 中。
### 运行 Torch Serve
Torch serve 提供了一个易于使用的命令行界面，并利用基于 REST 的 API 处理状态预测请求。用户可以从命令行运行它。此命令行调用接受您要服务的单个或多个模型，以及控制端口、主机和日志记录的其他可选参数。Torch Serve 支持运行自定义服务来处理特定的推理管理逻辑。
```
$ torchserve --help
usage: torchserve [-h] [-v | --version]
                          [--start]
                          [--stop]
                          [--ts-config TS_CONFIG]
                          [--model-store MODEL_STORE]
                          [--workflow-store WORKFLOW_STORE]
                          [--models MODEL_PATH1 MODEL_NAME=MODEL_PATH2... [MODEL_PATH1 MODEL_NAME=MODEL_PATH2... ...]]
                          [--log-config LOG_CONFIG]

torchserve

optional arguments:
  -h, --help            show this help message and exit
  -v, --version         Return TorchServe Version
  --start               Start the model-server
  --stop                Stop the model-server
  --ts-config TS_CONFIG
                        Configuration file for TorchServe
  --model-store         MODEL_STORE
                        Model store location where models can be loaded.
                        It is required if "model_store" is not defined in config.properties.
  --models MODEL_PATH1 MODEL_NAME=MODEL_PATH2... [MODEL_PATH1 MODEL_NAME=MODEL_PATH2... ...]
                        Models to be loaded using [model_name=]model_location
                        format. Location can be a HTTP URL, a model archive
                        file or directory contains model archive files in
                        MODEL_STORE.
  --log-config LOG_CONFIG
                        Log4j configuration file for TorchServe
  --ncs, --no-config-snapshots         
                        Disable snapshot feature
  --workflow-store WORKFLOW_STORE
                        Workflow store location where workflow can be loaded. Defaults to model-store
```

Torch Serve常用命令行及源代码：

- Torch Serve主函数

过 [ModelServer.java](https://github.com/pytorch/serve/blob/master/frontend/server/src/main/java/org/pytorch/serve/ModelServer.java#L84) 下 main()函数是程序入口点，该方法接受命令行参数，并调用相关方法。
```java
public static void main(String[] args) {
        //创建options对象，包含所有可用命令行参数
        Options options = ConfigManager.Arguments.getOptions();
        try {
            //解析命令行参数
            DefaultParser parser = new DefaultParser();
            CommandLine cmd = parser.parse(options, args, null, false);
            //初始化配置管理器，创建实例
            ConfigManager.Arguments arguments = new ConfigManager.Arguments(cmd);
            ConfigManager.init(arguments);
            ConfigManager configManager = ConfigManager.getInstance();
            //初始化插件管理器
            PluginsManager.getInstance().initialize();
            //初始化指标缓存
            MetricCache.init();
            //设置日志框架
            InternalLoggerFactory.setDefaultFactory(Slf4JLoggerFactory.INSTANCE);
            //创建 modelServer 对象
            ModelServer modelServer = new ModelServer(configManager);

            //钩子 JVM 停止，停止模型服务器
            Runtime.getRuntime()
                    .addShutdownHook(
                            new Thread() {
                                @Override
                                public void run() {
                                    modelServer.stop();
                                }
                            });
        	//启动Torch Serve 等待关闭
            modelServer.startAndWait();
        } catch (IllegalArgumentException e) {
            System.out.println("Invalid configuration: " + e.getMessage()); // NOPMD
        } catch (ParseException e) {
            HelpFormatter formatter = new HelpFormatter();
            formatter.setLeftPadding(1);
            formatter.setWidth(120);
            formatter.printHelp(e.getMessage(), options);
        } catch (Throwable t) {
            t.printStackTrace(); // NOPMD
        } finally {
            System.exit(1); //所有代码执行完，关闭JVM
        }
    }
```

- Torch Serve的启用

Torch serve的启用和主要通过 [ModelServer.java](https://github.com/pytorch/serve/blob/master/frontend/server/src/main/java/org/pytorch/serve/ModelServer.java#L84) 下的 startAndWait() 函数实现。
```java
public void startAndWait()
    throws InterruptedException, IOException, GeneralSecurityException,
    InvalidSnapshotException {
        try {
            //启动 REST 服务器，处理推理请求、管理请求和（如果启用）度量请求
            //推理和管理请求各开一个服务器，但是公享一组事件循环处理线程
            //一个事件循环线程中有请求接受(serverGroup)和请求处理(workerGroup)两个EventLoop
            //多个服务器，一套事件循环线程。
            //serverGroup主要负责监听新的连接，然后将这些连接分配给workerGroup进行处理。workerGroup则负责具体的I/O操作和连接管理。
            List<ChannelFuture> channelFutures = startRESTserver();
    
            startGRPCServers();
    
            // 是否禁用指标参数
            if (!configManager.isSystemMetricsDisabled()) {
                MetricManager.scheduleMetrics(configManager);
            }
    
            System.out.println("Model server started."); // NOPMD

            //开始等待和执行第一个 REST API 操作
            channelFutures.get(0).sync();
        } catch (InvalidPropertiesFormatException e) {
            logger.error("Invalid configuration", e);
        } finally {
            serverGroups.shutdown(true);
            logger.info("Torchserve stopped.");
        }
    }
```

- Torch Serve的停用

Torch serve的停用和主要通过 [ModelServer.java](https://github.com/pytorch/serve/blob/master/frontend/server/src/main/java/org/pytorch/serve/ModelServer.java#L84) 下的 stop() 函数实现。
```java
 public void stop() {
         //检查是否已经停止
        if (stopped.get()) {
            return;
        }

        stopped.set(true);
    	
     	//停止服务器
        stopgRPCServer(inferencegRPCServer);
        stopgRPCServer(managementgRPCServer);

        for (ChannelFuture future : futures) {
            try {
                future.channel().close().sync();
            } catch (InterruptedException ignore) {
                ignore.printStackTrace();
            }
        }

        SnapshotManager.getInstance().saveShutdownSnapshot();

        //停止接受和处理请求的线程
        serverGroups.shutdown(true);
        serverGroups.init();

        try {
            //将模型的所有版本从模型管理实例中注销
            exitModelStore();
        } catch (Exception e) {
            e.printStackTrace(); // NOPMD
        }
    }
}
```

时序图：
![image.png](https://cdn.nlark.com/yuque/0/2023/png/35887099/1684795121749-a1ec25a3-5a1a-428f-9f48-8c61dd1b2720.png#averageHue=%23f1f1f1&clientId=u2978bd1a-44d2-4&from=paste&height=594&id=uc4765cbe&originHeight=1196&originWidth=1198&originalType=binary&ratio=2&rotation=0&showTitle=false&size=189579&status=done&style=none&taskId=ueb7eb53b-2aae-466d-8858-b8cfdaaa039&title=&width=595)
### Manage API
Torch Serve 可以在启动时加载模型，设置模型参数，同时也可以在运行时通过 Manage API 进行动态管理，而不需要停止和重启。Management API 在端口 8081 上侦听，默认情况下只能从本地主机访问。

Torch Serve 会在启动 REST 服务器的时候初始化模型存储器，并调用 [ModelManager.java](https://github.com/pytorch/serve/blob/master/frontend/server/src/main/java/org/pytorch/serve/wlm/ModelManager.java) 管理模型。ModelManager 这个类的实例在运行时只能有一个，但可以管理多个模型。这样做的主要目的保证所有对模型的操作（加载模型、卸载模型、获取模型信息等）都通过同一个 ModelManager 实例进行，以便对多个模型进行统一和集中的管理。
在调用模型管理 API 中有几个重要的 JAVA 文件：
[ConfigManager.java](https://github.com/pytorch/serve/blob/master/frontend/server/src/main/java/org/pytorch/serve/util/ConfigManager.java#L289)：管理和提供访问 Torch Serve 的配置信息，这些配置信息可以来自不同的来源，例如环境变量，属性文件，或者命令行参数。
[ModelManager.java](https://github.com/pytorch/serve/blob/master/frontend/server/src/main/java/org/pytorch/serve/wlm/ModelManager.java): 这是模型管理的核心类，负责模型的加载、卸载、更新等操作。当一个模型管理 API 调用发生时，该类会被调用。

常用管理命令和相关代码：

- 注册模型
```
curl -X POST  "http://localhost:8081/models?url=https://torchserve.pytorch.org/sse-test/squeezenet1_1.mar&s3_sse_kms=true"

{
  "status": "Model \"squeezenet_v1.1\" Version: 1.0 registered with 0 initial workers. Use scale workers API to add workers for the model."
}

'''
POST /models
url - Model archive download url. Supports the following locations:
	a local model archive (.mar); the file must be in the model_store folder (and not in a subfolder).
	a URI using the HTTP(s) protocol. TorchServe can download .mar files from the Internet.
model_name - the name of the model; this name will be used as {model_name} in other APIs as part of the path. If this parameter is not present, modelName in MANIFEST.json will be used.
handler - the inference handler entry-point. This value will override handler in MANIFEST.json if present. NOTE: Make sure that the given handler is in the PYTHONPATH. The format of handler is module_name:method_name.
runtime - the runtime for the model custom service code. This value will override runtime in MANIFEST.json if present. The default value is PYTHON.
batch_size - the inference batch size. The default value is 1.
max_batch_delay - the maximum delay for batch aggregation. The default value is 100 milliseconds.
initial_workers - the number of initial workers to create. The default value is 0. TorchServe will not run inference until there is at least one work assigned.
synchronous - whether or not the creation of worker is synchronous. The default value is false. TorchServe will create new workers without waiting for acknowledgement that the previous worker is online.
response_timeout - If the model's backend worker doesn't respond with inference response within this timeout period, the worker will be deemed unresponsive and rebooted. The units is seconds. The default value is 120 seconds.
3_sse_kms - encrypted

```
```java
// public ModelArchive(Manifest manifest, String url, File modelDir, boolean extracted) {
//     this.manifest = manifest;
//     this.url = url;
//     this.modelDir = modelDir;
//     this.extracted = extracted;
//     this.modelConfig = null;
// }


public ModelArchive registerModel(
            String url,
            String modelName,
            Manifest.RuntimeType runtime,
            String handler,
            int batchSize,
            int maxBatchDelay,
            int responseTimeout,
            String defaultModelName,
            boolean ignoreDuplicate,
            boolean isWorkflowModel,
            boolean s3SseKms)
            throws ModelException, IOException, InterruptedException, DownloadArchiveException {

        ModelArchive archive;
        //是否为workflow
        if (isWorkflowModel && url == null) { 
            Manifest manifest = new Manifest();
            manifest.getModel().setVersion("1.0");
            manifest.getModel().setModelVersion("1.0");
            manifest.getModel().setModelName(modelName);
            manifest.getModel().setHandler(new File(handler).getName());
            manifest.getModel().setEnvelope(configManager.getTsServiceEnvelope());
            File f = new File(handler.substring(0, handler.lastIndexOf(':')));
            archive = new ModelArchive(manifest, url, f.getParentFile(), true);
        } else {
            //根据mar 文件地址创建一个modelarchive 实例
            //解压到临时目录中，模型加载完成后目录被删除
            archive =
                    createModelArchive(
                            modelName, url, handler, runtime, defaultModelName, s3SseKms);
        }

        Model tempModel =
                createModel(archive, batchSize, maxBatchDelay, responseTimeout, isWorkflowModel);
    	//没version默认为1.0
        String versionId = archive.getModelVersion();

        try {
            createVersionedModel(tempModel, versionId);
        } catch (ConflictStatusException e) {
            if (!ignoreDuplicate) {
                throw e;
            }
        }

        setupModelDependencies(tempModel);

        logger.info("Model {} loaded.", tempModel.getModelName());

        return archive;
    }

```

- 增加/减少特定模型的工人数量
```
curl -v -X PUT "http://localhost:8081/models/noop?min_worker=3"

< HTTP/1.1 202 Accepted
< content-type: application/json
< x-request-id: 42adc58e-6956-4198-ad07-db6c620c4c1e
< content-length: 47
< connection: keep-alive
< 
{
  "status": "Processing worker updates..."
}

'''
PUT /models/{model_name}/{version}

min_worker - (optional) the minimum number of worker processes. TorchServe will try to maintain this minimum for specified model. The default value is 1.
max_worker - (optional) the maximum number of worker processes. TorchServe will make no more that this number of workers for the specified model. The default is the same as the setting for min_worker.
synchronous - whether or not the call is synchronous. The default value is false.
timeout - the specified wait time for a worker to complete all pending requests. If exceeded, the work process will be terminated. Use 0 to terminate the backend worker process immediately. Use -1 to wait infinitely. The default value is -1.
'''

#The asynchronous call returns with HTTP code 202 before trying to create workers.
#The synchronous call returns with HTTP code 200 after all workers have been adjusted.
```
```java
pupblic CompletableFuture<Integer> updateModel(
            String modelName,
            String versionId,
            int minWorkers,
            int maxWorkers,
            boolean isStartup,
            boolean isCleanUp)
            throws ModelVersionNotFoundException, WorkerInitializationException {
        Model model = getVersionModel(modelName, versionId);

        if (model == null) {
            throw new ModelVersionNotFoundException(
                    "Model version: " + versionId + " does not exist for model: " + modelName);
        }
    	//如果设备类型为GPU,检验是否能运行
        if (model.getParallelLevel() > 1 && model.getDeviceType() == ModelConfig.DeviceType.GPU) {
            //计算当前GPU支持的最大运行级别
            int capacity = model.getNumCores() / model.getParallelLevel();
            if (capacity == 0) {
                logger.error(
                        "there are no enough gpu devices to support this parallelLever: {}",
                        model.getParallelLevel());
                throw new WorkerInitializationException(
                        "No enough gpu devices for model:"
                                + modelName
                                + " parallelLevel:"
                                + model.getParallelLevel());
            } else {
                minWorkers = minWorkers > capacity ? capacity : minWorkers;
                maxWorkers = maxWorkers > capacity ? capacity : maxWorkers;
                logger.info(
                        "model {} set minWorkers: {}, maxWorkers: {} for parallelLevel: {} ",
                        modelName,
                        minWorkers,
                        maxWorkers,
                        model.getParallelLevel());
            }
        }
        //设置worker数
        model.setMinWorkers(minWorkers);
        model.setMaxWorkers(maxWorkers);
        logger.debug("updateModel: {}, count: {}", modelName, minWorkers);

        //更新工作负载管理器，调用wlm中的addThreads添加或停止workerthread
        return wlm.modelChanged(model, isStartup, isCleanUp);
    }

```

- 描述模型状态
```
curl http://localhost:8081/models/noop
[
    {
      "modelName": "noop",
      "modelVersion": "1.0",
      "modelUrl": "noop.mar",
      "engine": "Torch",
      "runtime": "python",
      "minWorkers": 1,
      "maxWorkers": 1,
      "batchSize": 1,
      "maxBatchDelay": 100,
      "workers": [
        {
          "id": "9000",
          "startTime": "2018-10-02T13:44:53.034Z",
          "status": "READY",
          "gpu": false,
          "memoryUsage": 89247744
        }
      ]
    }
]

'''
GET /models/{model_name}/{version}
'''
```

- 注销模型
```
curl -X DELETE http://localhost:8081/models/noop/1.0

{
  "status": "Model \"noop\" unregistered"
}

'''
DELETE /models/{model_name}/{version}
'''
```
```java
public int unregisterModel(String modelName, String versionId, boolean isCleanUp) {
        ModelVersionedRefs vmodel = modelsNameMap.get(modelName);
        if (vmodel == null) {
            logger.warn("Model not found: " + modelName);
            return HttpURLConnection.HTTP_NOT_FOUND;
        }

        if (versionId == null) {
            versionId = vmodel.getDefaultVersion();
        }

        Model model;
        int httpResponseStatus;

        try {
            //删除版本号，将 Worker 设置为0
            model = vmodel.removeVersionModel(versionId);
            model.setMinWorkers(0);
            model.setMaxWorkers(0);
            //告诉 Work Load Manage模型发生改变
            CompletableFuture<Integer> futureStatus = wlm.modelChanged(model, false, isCleanUp);
            httpResponseStatus = futureStatus.get();

            // Only continue cleaning if resource cleaning succeeded

            if (httpResponseStatus == HttpURLConnection.HTTP_OK) {
                //从模型的临时目录下清理模型
                model.getModelArchive().clean();
                startupModels.remove(modelName);
                logger.info("Model {} unregistered.", modelName);
            } else {
                if (versionId == null) {
                    versionId = vmodel.getDefaultVersion();
                }
                vmodel.addVersionModel(model, versionId);
            }

            if (vmodel.getAllVersions().size() == 0) {
                modelsNameMap.remove(modelName);
            }

            if (!isCleanUp && model.getModelUrl() != null) {
                ModelArchive.removeModel(configManager.getModelStore(), model.getModelUrl());
            }
        } catch (ModelVersionNotFoundException e) {
            logger.warn("Model {} version {} not found.", modelName, versionId);
            httpResponseStatus = HttpURLConnection.HTTP_BAD_REQUEST;
        } catch (InvalidModelVersionException e) {
            logger.warn("Cannot remove default version {} for model {}", versionId, modelName);
            httpResponseStatus = HttpURLConnection.HTTP_FORBIDDEN;
        } catch (ExecutionException | InterruptedException e1) {
            logger.warn("Process was interrupted while cleaning resources.");
            httpResponseStatus = HttpURLConnection.HTTP_INTERNAL_ERROR;
        }

        return httpResponseStatus;
    }
```

- 列出注册模型
```
curl "http://localhost:8081/models?limit=2&next_page_token=2"

{
  "nextPageToken": "4",
  "models": [
    {
      "modelName": "noop",
      "modelUrl": "noop-v1.0"
    },
    {
      "modelName": "noop_v0.1",
      "modelUrl": "noop-v0.1"
    }
  ]
}

'''
GET /models

limit - (optional) the maximum number of items to return. It is passed as a query parameter. The default value is 100.
next_page_token - (optional) queries for next page. It is passed as a query parameter. This value is return by a previous API call.
'''
```

- 查看可用管理API
```
curl -X OPTIONS http://localhost:8081

'''
OPTIONS /
'''
```

- 设置模型默认版本号码
```
curl -v -X PUT http://localhost:8081/models/noop/2.0/set-default

'''
PUT /models/{model_name}/{version}/set-default
'''
```

### Inference API
推理 API 在端口 8080 上侦听，默认情况下只能从本地主机访问。

在调用模型推理 API 中有几个重要的 JAVA 文件：
[WorkLoadManager.java](https://github.com/pytorch/serve/blob/master/frontend/server/src/main/java/org/pytorch/serve/wlm/WorkLoadManager.java#L23): 用来管理加载的模型，负责管理工作线程和处理推理请求。可以过该类查看 Worker 数量和家康状况。
[WorkerThread.java](https://github.com/pytorch/serve/blob/master/frontend/server/src/main/java/org/pytorch/serve/wlm/WorkerThread.java): 用来管理进程和进程池。每个工作线程在后台运行，等待处理传入的推理请求。
WorkerLifeCycle.java: 此类定义了模型的工作进程的生命周期。它包含启动、停止工作进程的代码，以及监控工作进程的日志输出的代码。

```
curl -X OPTIONS http://localhost:8080
```
```
curl http://localhost:8080/ping
{
  "status": "Healthy"
}
```
```
curl http://localhost:8080/predictions/resnet-18 -T kitten_small.jpg
curl http://localhost:8080/predictions/resnet-18 -F "data=@kitten_small.jpg"

#multiple input
curl http://localhost:8080/predictions/squeezenet1_1 -F 'data=@docs/images/dogs-before.jpg' -F 'data=@docs/images/kitten_small.jpg'

import requests
res = requests.post("http://localhost:8080/predictions/squeezenet1_1", files={'data': open('docs/images/dogs-before.jpg', 'rb'), 'data': open('docs/images/kitten_small.jpg', 'rb')})

'''
POST /predictions/{model_name}/{version}
'''
```
```
#Torchserve makes use of Captum's functionality to return the explanations of the models that is served.
curl http://127.0.0.1:8080/explanations/mnist -T examples/image_classifier/mnist/test_data/0.png
```

### 轮询批处理
每一个模型可以有多个 Worker，其数量限制在 minWorke 和 maxWorker 之间。 一般情况下每一个 Worker 会分配一个 WorkerThread。同时每一个工作线程会初始化一个 BatchAggregator 对象，用于存放批量处理任务。 所有模型的线程被放到一个名为 backendGroup 的线程池中。
```java
public void pollBatch(String threadId, long waitTime, Map<String, Job> jobsRepo)
throws InterruptedException {
    //检查jobsRepo和threadId是否有效，
    if (jobsRepo == null || threadId == null || threadId.isEmpty()) {
        throw new IllegalArgumentException("Invalid input given provided");
    }

    if (!jobsRepo.isEmpty()) {
        throw new IllegalArgumentException(
            "The jobs repo provided contains stale jobs. Clear them!!");
    }

    //定义jobsQueue，并从jobDb中获取threadId
    LinkedBlockingDeque<Job> jobsQueue = jobsDb.get(threadId);
    //尝试从jobsQueue中取出一个任务并放入jobsRepo中
    if (jobsQueue != null && !jobsQueue.isEmpty()) {
        Job j = jobsQueue.poll(waitTime, TimeUnit.MILLISECONDS);
        if (j != null) {
            jobsRepo.put(j.getJobId(), j);
            return;
        }
    }
	
    try {
        //尝试获取一个锁
        lock.lockInterruptibly();
        long maxDelay = maxBatchDelay;
        jobsQueue = jobsDb.get(DEFAULT_DATA_QUEUE);

        Job j = jobsQueue.poll(Long.MAX_VALUE, TimeUnit.MILLISECONDS);
        logger.trace("get first job: {}", Objects.requireNonNull(j).getJobId());

        
        jobsRepo.put(j.getJobId(), j);
        // batch size always is 1 for describe request job and stream prediction request job
        if (j.getCmd() == WorkerCommands.DESCRIBE
            || j.getCmd() == WorkerCommands.STREAMPREDICT) {
            return;
        }
        long begin = System.currentTimeMillis();
        //尝试获取batchSize - 1个任务
        for (int i = 0; i < batchSize - 1; ++i) {
            j = jobsQueue.poll(maxDelay, TimeUnit.MILLISECONDS);
            if (j == null) {
                break;
            }
            long end = System.currentTimeMillis();
            //如果任务的命令是DESCRIBE或STREAMPREDICT，那么就将任务放回到队列中并退出循环
            if (j.getCmd() == WorkerCommands.DESCRIBE
                || j.getCmd() == WorkerCommands.STREAMPREDICT) {
                // Add the job back into the jobsQueue
                jobsQueue.addFirst(j);
                break;
            }
            maxDelay -= end - begin;
            begin = end;
            if (j.getPayload().getClientExpireTS() > System.currentTimeMillis()) {
                jobsRepo.put(j.getJobId(), j);
            } else {
                logger.warn(
                    "Drop inference request {} due to client timeout",
                    j.getPayload().getRequestId());
            }
            //任务超时，丢弃任务
            if (maxDelay <= 0) {
                break;
            }
        }
        logger.trace("sending jobs, size: {}", jobsRepo.size());
    } finally {
        //释放锁
        if (lock.isHeldByCurrentThread()) {
            lock.unlock();
        }
    }
}

```

pollBatch() 函数中会进行进行轮询的操作，它是从工作队列中获取一批（batch）请求。这是一个阻塞操作，意味着如果队列中没有足够的请求，这个函数会等待，直到有足够的请求可以形成一个批次。默认的批次大小为1，可以通过配置文件设置。
```java
public void run() {
    responseTimeout = model.getResponseTimeout();
    Thread thread = Thread.currentThread();
    thread.setName(getWorkerName());
    currentThread.set(thread);
    BaseModelRequest req = null;
    int status = HttpURLConnection.HTTP_INTERNAL_ERROR;

    try {
        //链接后端
        connect();

        while (isRunning()) {
            //聚合器
            //读取聚合器中的请求，如果是控制命令则直接返回一个ModelLoadModelRequest
            //一次只能处理一个控制命令（load，unload）
            //如果不是则将模型加入到ModelInferenceRequest
            req = aggregator.getRequest(workerId, state);

            long wtStartTime = System.currentTimeMillis();
            logger.info("Flushing req.cmd {} to backend at: {}", req.getCommand(), wtStartTime);
            //确定所需channel数
            //如果请求的命令是加载（LOAD）或预测（PREDICT、STREAMPREDICT），并且模型的并行级别大于1且并行类型不是PP，
            //那么请求数量就是模型的并行级别，否则就是1
            int repeats =
            (req.getCommand() == WorkerCommands.LOAD)
            || ((req.getCommand() == WorkerCommands.PREDICT
                 || req.getCommand()
                 == WorkerCommands.STREAMPREDICT)
                && model.getParallelLevel() > 1
                && model.getParallelType()
                != ModelConfig.ParallelType.PP)
            ? model.getParallelLevel()
            : 1;
            //把每个请求发给后端
            for (int i = 0; backendChannel.size() > 0 && i < repeats; i++) {
                backendChannel.get(i).writeAndFlush(req).sync();
            }
        	//是否为流式预测
            boolean isStreaming =
            req.getCommand() == WorkerCommands.STREAMPREDICT ? true : false;
            //接受后端响应的变量
            ModelWorkerResponse reply = null;

            boolean jobDone = false;
            long totalDuration = 0;
            do {
                long begin = System.currentTimeMillis();
                for (int i = 0; i < repeats; i++) {
                    //响应队列里获取响应，检查是哪种命令
                    reply = replies.poll(responseTimeout, TimeUnit.SECONDS);
                }

                long duration = System.currentTimeMillis() - begin;
            	//有reply
                if (reply != null) {
                    //响应发给聚合器
                    jobDone = aggregator.sendResponse(reply);
                    logger.debug("sent a reply, jobdone: {}", jobDone);
                    //失败且不是描述请求
                } else if (req.getCommand() != WorkerCommands.DESCRIBE) {
                    int val = model.incrFailedInfReqs();
                    logger.error("Number or consecutive unsuccessful inference {}", val);
                    throw new WorkerInitializationException(
                        "Backend worker did not respond in given time");
                }
                totalDuration += duration;
            } while (!jobDone);
            logger.info("Backend response time: {}", totalDuration);

            //对不同请求错误的处理方法
            switch (req.getCommand()) {
                case PREDICT:
                    model.resetFailedInfReqs();
                    break;
                case STREAMPREDICT:
                    model.resetFailedInfReqs();
                    break;
                case LOAD:
                    if (reply.getCode() == 200) {
                    setState(WorkerState.WORKER_MODEL_LOADED, HttpURLConnection.HTTP_OK);
                    backoffIdx = 0;
            } else {
                    setState(WorkerState.WORKER_ERROR, reply.getCode());
                    status = reply.getCode();
            }
                    break;
                case DESCRIBE:
                    if (reply == null) {
                    aggregator.sendError(
                    req,
                    "Failed to get customized model matadata.",
                    HttpURLConnection.HTTP_INTERNAL_ERROR);
            }
                    break;
                case UNLOAD:
                case STATS:
                default:
                    break;
            }
                    req = null;
                    double workerThreadTime =
                    (System.currentTimeMillis() - wtStartTime) - totalDuration;
                    if (this.workerThreadTimeMetric != null) {
                    try {
                    this.workerThreadTimeMetric.addOrUpdate(
                    this.workerThreadTimeMetricDimensionValues, workerThreadTime);
            } catch (Exception e) {
                    logger.error("Failed to update frontend metric WorkerThreadTime: ", e);
            }
            }
            }
            } catch (InterruptedException e) {
                    logger.debug("System state is : " + state);
                    if (state == WorkerState.WORKER_SCALED_DOWN || state == WorkerState.WORKER_STOPPED) {
                    logger.debug("Shutting down the thread .. Scaling down.");
            } else {
                    logger.debug(
                    "Backend worker monitoring thread interrupted or backend worker process died.",
                    e);
            }
            } catch (WorkerInitializationException e) {
                    logger.error("Backend worker error", e);
            } catch (OutOfMemoryError oom) {
                    logger.error("Out of memory error when creating workers", oom);
                    status = HttpURLConnection.HTTP_ENTITY_TOO_LARGE;
                    if (java.lang.System.getenv("SM_TELEMETRY_LOG") != null) {
                    loggerTelemetryMetrics.info(
                    "ModelServerError.Count:1|#TorchServe:{},{}:-1",
                    ConfigManager.getInstance().getVersion(),
                    oom.getClass().getCanonicalName());
            }
            } catch (Throwable t) {
                    logger.warn("Backend worker thread exception.", t);
                    if (java.lang.System.getenv("SM_TELEMETRY_LOG") != null) {
                    loggerTelemetryMetrics.info(
                    "ModelServerError.Count:1|#TorchServe:{},{}:-1",
                    ConfigManager.getInstance().getVersion(),
                    t.getClass().getCanonicalName());
            }
            } finally {
                    // WorkerThread is running in thread pool, the thread will be assigned to next
                    // Runnable once this worker is finished. If currentThread keep holding the reference
                    // of the thread, currentThread.interrupt() might kill next worker.
                    //关闭所有channel
        			for (int i = 0; backendChannel.size() > 0 && i < model.getParallelLevel(); i++) {
                    backendChannel.get(i).disconnect();
            }
                    //将线程引用设为null,防止干扰下一个thread
                    currentThread.set(null);
                    Integer exitValue = lifeCycle.getExitValue();

                    //占用过多内存
                    if (exitValue != null && exitValue == 137) {
                    status = HttpURLConnection.HTTP_ENTITY_TOO_LARGE;
            }
                	//请求未完成
                    if (req != null) {
                    aggregator.sendError(req, "Worker died.", status);
            }
                    //停止worker，修改状态
                    setState(WorkerState.WORKER_STOPPED, status);
                    lifeCycle.exit();
                    if (isHealthy()) { // still within maxRetryTimeoutInMill window
                    retry();
            }
            }
            }
```

时序图：
### ![image.png](https://cdn.nlark.com/yuque/0/2023/png/35887099/1684831271359-f983532d-998e-4c32-b380-beecf214fff4.png#averageHue=%23f7f7f6&clientId=u2978bd1a-44d2-4&from=paste&height=561&id=u5fddbcae&originHeight=1122&originWidth=758&originalType=binary&ratio=2&rotation=0&showTitle=false&size=135814&status=done&style=none&taskId=ucdf11698-4a18-42ee-af65-429fd23ac88&title=&width=379)

### API 处理时序图
![image.png](https://cdn.nlark.com/yuque/0/2023/png/35887099/1685321504098-b32c7ea4-c246-45fa-9437-8f456dd2ea56.png#averageHue=%23f6f6f6&clientId=u795b0598-2998-4&from=paste&height=531&id=ub919cfc2&originHeight=1062&originWidth=1134&originalType=binary&ratio=2&rotation=0&showTitle=false&size=145404&status=done&style=none&taskId=u7b9e5d55-faf7-49c5-9365-71d727a6007&title=&width=567)
### Work flow
工作流将 Pytorch 模型或函数组成的集合包装进 .war(workflow-archive) 文件。工作流可以表示为 DAG，其中的节点可以是存储在 .mar 文件中的模型，或者是存储在 handler 文件中的函数。顺序和并行管道都适用。
```
# sequential pipeline
input -> function1 -> model1 -> model2 -> function2 -> output


#parallel pipeline
                          model1
                         /       \
input -> preprocessing ->         -> aggregate_func
                         \       /
                          model2
```
工作流程规范文件(YAML)
```
#models which include global model parameters
#m1,m2,m3 all the relevant model parameters which would override the global model params
#dag which describes the structure of the workflow, which nodes feed into which other nodes


models:
    #global model params 
    min-workers: 1 #Number of minimum workers launched for every workflow model
    max-workers: 4 #Number of maximum workers launched for every workflow model
    batch-size: 3 #Batch size used for every workflow model
    max-batch-delay : 5000 #Maximum batch delay time TorchServe waits for every workflow model to receive batch_size number of requests
    retry-attempts : 3 #Retry attempts for a specific workflow node in case of a failure
    timeout-ms : 5000 #Timeout in MilliSeconds for a given node
    m1:
       url : model1.mar #local or public URI
       min-workers: 1   #override the global params
       max-workers: 2
       batch-size: 4
     
    m2:
       url : model2.mar

    m3:
       url : model3.mar
       batch-size: 3

    m4:
      url : model4.mar
 
dag:
  pre_processing : [m1]
  m1 : [m2]
  m2 : [m3]
  m3 : [m4]
  m4 : [postprocessing]
```
*每个工作流以字节的形式接收输入
*工作流仅支持 String、Int、List、Dict of String、int、Json serializable objects、byte array 和 Torch Tensors输出
*不支持 API 更改工作流程，需要取消注册重新注册更改。同时不支持快照和版本号
*工作流中不能出现已经注册的公共模型
```
#archiver model(mat)
#archiver workflow(war)
torch-workflow-archiver -f --workflow-name WORKFLOW_NAME --spec-file YAML_FILE --handler HANDLER_PY_FILE --export-path wf_store/

# torchserve --start

#Management API
#register POST /workflows
curl -X POST  "http://localhost:8081/workflows?url=https://<public_url>/myworkflow.mar"
#describe  GET /workflows/{workflow_name}
curl http://localhost:8081/workflows/myworkflow
#unregister DELETE /workflows/{workflow_name}
curl -X DELETE http://localhost:8081/workflows/workflow_name
#list GET /models
curl "http://localhost:8081/workflows"



#Inference API
#prdict POST /wfpredict/{workflow_name}
curl http://localhost:8080/wfpredict/myworkflow -T kitten_small.jpg
```

## Torch Serve 配置

可以通过三种方式配置 Torch Serve。按照优先顺序，它们是：

1. 环境变量
2. 命令行参数
3. 配置文件

Torch Serve 支持两种类型的配置：文件配置和命令行配置。文件配置通过 YAML 格式的文件进行定义，命令行配置通过命令行参数进行定义。下面是 Torch Serve 支持的文件配置和命令行主要配置及其功能：

### 配置文件 (config.properties)
Torch Serve 使用一个 config.properties 文件来存储配置。Torch Serve 按优先顺序使用以下内容来定位此 config.properties 文件：

- TS_CONFIG_FILE环境变量。
- --ts-config命令行参数。
- 模型所在文件夹，Torch Serve 会从当前工作目录加载该 config.properties。
- 如果以上均未指定，Torch Serve 将加载具有默认值的内置配置。

主要参数:

1. 监听端口
- inference_address: 推理 REST API 的地址，默认值为 http://127.0.0.1:8080
- management_address: 管理 REST API的地址，默认值为 http://127.0.0.1:8081
- grpc_inference_port: 推理 gRPC API 的绑定端口，默认值: 7070
- grpc_management_port: management gRPC API 的绑定端口，默认值: 7071
- metrics_address: 指标 REST API 的地址，默认值为http://127.0.0.1:8082
2. 模型加载
- load_models: 用于设置在启动时需要加载的模型：
   - standalone: 默认值：N/A，没有模型在启动时加载。
   - all: 加载 model_store里所有的模型。
   - model1.mar:  加载具有特殊名称的模型。
   - model1=model1.mar, model2=model2.mar: 给特殊模型一个具体名称。
- model_store: 用于设置模型存储的路径：
   - standalone: 默认值：N/A，禁止从本地磁盘加载模型。
   - pathname: 模型存储的位置。
3. 模型：

除了在模型存档 (.mar 文件) 中添加 MAR-INF/config.properties 来配置与特定模型相关的参数。也可在torch serve 的 config.properties 文件中设置特定于模型的配置。该值以 json 格式显示：
```json
models={\
  "noop": {\                        //模型名称
    "1.0": {\                       //模型版本号
        "defaultVersion": true,\    //是否为默认版本
        "marName": "noop.mar",\     //模型存档名称
        "minWorkers": 1,\           //模型最小 Worker 数
        "maxWorkers": 1,\           //模型最大 Worker 数
        "batchSize": 4,\            //模型批量大小，默认为1
        "maxBatchDelay": 100,\      //模型批量的最大延迟，以毫秒为单位，默认100
        "responseTimeout": 120\     //特定模型响应的超时秒数，默认120
    }\
  }
}
```

4. 其它
- number_of_netty_threads: 用于设置 Netty 服务器的线程数，指定了 Netty 服务器中 EventLoopGroup 的线程数，默认为 JVM 可用的逻辑处理器数。
- job_queue_size: 在后端可以服务之前前端将排队的推理作业数。默认值：100。
- install_py_dep_per_model: 模型服务器是否使用 requirements.txt 中的 Python 包列表，默认值为 False。
- default_workers_per_model: 启动时加载的每个模型创建的 Worker 数。默认值为系统中可用的 GPU 或 JVM 可用的逻辑处理器数。


### 命令行参数
可以在初始化 Torch Serve 时通过命令行配置：

- --ts-config：如果未设置环境变量 TS_CONFIG_FILE，Torch Serve 需要加载的指定配置文件。
- --model-store：覆盖 config.properties 文件中 model_store 属性。
- --models：覆盖 sconfig.properties 文件中 load_model 属性。
- --log-config：覆盖默认的 log4j2.xml。
- --foreground：Torch Serve 是否在在前台运行。禁用时为后台运行。


更多配置信息可查看 [Advanced configuration](https://github.com/pytorch/serve/blob/master/docs/configuration.md)。

## Torch serve 从 S3 下载文件
### 现有方法

1. 直接使用

在TorchServe中，可以直接从 S3 中拉取封装好的模型（.mar 文件）。
1. 启动一个 Torch Serve 实例，修改配置文件。在配置文件中，设置 `s3_snapshot_download_enabled=true` 来允许从 S3 下载模型
2. 在 Torch Serve 启动后，用户可以通过调用 REST API 来注册模型。在这个请求中，用户需要提供指向 S3 中 .mar 文件的 URL。
```groovy
curl -v -X POST "http://localhost:8081/models?url=s3://mybucket/mymodel.mar"
```
该命令会在本地的 model_store 文件下建立一个同名的模型文件，当用户再次利用 REST API 注销模型时，本地文件下的同名文件会被删除。
如果 S3 中有加密服务，用户需要设置以下环境变量`AWS_ACCESS_KEY_ID`，`AWS_SECRET_ACCESS_KEY`和
`AWS_DEFAULT_REGION`。并在 URL 后添加`s3_sse_kms=true`来注册加密模型。
```groovy
curl -X POST  "http://localhost:8081/models?url=https://torchserve.pytorch.org/sse-test/squeezenet1_1.mar&s3_sse_kms=true"
```

模型拉取时序图：
![image.png](https://cdn.nlark.com/yuque/0/2023/png/35887099/1685044157697-b1fd90d7-7723-4787-a8f4-3331041b7238.png#averageHue=%23f5f5f4&clientId=u3d192c52-c5a8-4&from=paste&height=615&id=u1e40e8ba&originHeight=1230&originWidth=688&originalType=binary&ratio=2&rotation=0&showTitle=false&size=153439&status=done&style=none&taskId=u1c8ea160-6de5-4c86-9ee6-822b2d4e54d&title=&width=344)
在上述流程中 Torch Serve 首先通过 HttpUtils.java 中 httpUtils() 函数，利用请求头和java.net.HttpURLConnection 方法从 S3 服务器上读取返回的数据并利用 org.apache.commons.io.FileUtils 将读取的输入流复制到本地 model_store 中。其次通过 ModelArchive.java 中的 load() 函数对已经下载到本地的模型文件进行解析和加载并返回一个 ModelArchive 对象供后续调用。最后利用 ModelManage.java 的registerModel() 函数将模型添加到 modelsNameMap 进行统一管理。

2. 使用fsspec Python 库，直接通过pythoon管理不同文件系统 [stream_inference.py](https://github.com/pytorch/serve/blob/4450287d9a8322e5a670c5f473c37e192de92dc1/examples/cloud_storage_stream_inference/stream_inference.py)。
## 拓展
根据调研 Torch Serve 目前有两种可以编写插件的方法：在前端或后端集成。

在前端集成时 Torch Serve 在启动时可以设置插件路径(plugin_path)，根据 [model_server.py](https://github.com/pytorch/serve/blob/master/ts/model_server.py) 该参数会被添加到Java 的类路径中。这意味着 Java 的类加载器会从这个路径加载类，所以插件路需要指向 Java 编写的插件。

在后端集成时可以利用 python 编写自定义模型处理器。然后将自定义模型处理器与模型数据打包进同一个 .mar 文件。然后使用 Torch Serve API 对模型进行注册，以此来运行自定义的python脚本。

### 实现方法
以从 S3 拉取模型到 Torch Serve 为例进行插件编写：

1. JAVA 环境（前端）

Torch Serve 提供 [serving-sdk](https://github.com/pytorch/serve/tree/master/serving-sdk) 和 [plugins](https://github.com/pytorch/serve/tree/master/plugins) 供用户利用插件进行拓展。用户可以直接继承 [serving-sdk](https://github.com/pytorch/serve/tree/master/serving-sdk) 下的JAVA 类进行自定义操作。

- 在环境变量设置AWS_ACCESS_KEY_ID，AWS_SECRET_ACCESS_KEY，AWS_DEFAULT_REGION参数。
- 在 Torce Serve 的 plugin 文件夹下建立一个 s3 文件，在该目录下分别创建一个 build.gradle 文件和src/main 目录。再在 scr/main 下创建 resources/META-INF/services 目录和java/org/pytorch/serve/plugins/s3 目录。
- 在 java/org/pytorch/serve/plugins/s3 目录下创建一个 S3DownloaderEndpoint.java 文件。其通过 [serving-sdk](https://github.com/pytorch/serve/tree/master/serving-sdk) 继承 ModelServerEndpoint 类来创建一个端点插件类，并利用 S3 SDK 下载相应文件：
```java
package org.pytorch.serve.plugins.s3;

import org.pytorch.serve.servingsdk.ModelServerEndpoint;
import org.pytorch.serve.servingsdk.http.Request;
import org.pytorch.serve.servingsdk.http.Response;
import com.amazonaws.auth.AWSStaticCredentialsProvider;
import com.amazonaws.auth.BasicAWSCredentials;
import com.amazonaws.services.s3.AmazonS3;
import com.amazonaws.services.s3.AmazonS3ClientBuilder;
import com.amazonaws.services.s3.model.S3Object;
import java.io.InputStream;
import java.nio.file.StandardCopyOption;
import java.nio.file.Files;
import java.nio.file.Paths;

@Endpoint(
        urlPattern = "S3download",
        endpointType = EndpointTypes.MANAGEMENT,
        description = "download file from s3.")
public class S3DownloaderEndpoint extends ModelServerEndpoint {
    
    @Override
    public void doGet(Request req, Response rsp, Context ctx) throws IOException {
        rsp.setStatus(200);
        
        //从环境变量中获取密钥
        String awsAccessKey = System.getenv("AWS_ACCESS_KEY_ID");
        String awsSecretKey = System.getenv("AWS_SECRET_ACCESS_KEY");
        String regionName = System.getenv("AWS_DEFAULT_REGION");

         // 初始化访问凭证和 S3 客户端
        BasicAWSCredentials awsCreds = new BasicAWSCredentials(awsAccessKey, awsSecretKey);
        
        AmazonS3 s3Client = AmazonS3ClientBuilder.standard()
                                .withRegion(regionName)
                                .withCredentials(new AWSStaticCredentialsProvider(awsCreds))
                                .build();
        
        //文件名称和文件桶
        String bucketName = req.getParameter("bucket");
		String key = req.getParameter("key"); 
        
        S3Object s3object = s3Client.getObject(bucketName, key);
        InputStream is = s3object.getObjectContent();
        
        Files.copy(is, Paths.get("/path/local_file"), StandardCopyOption.REPLACE_EXISTING);
        rsp.getOutputStream().write(("File downloaded successfully to file_path").getBytes(StandardCharsets.UTF_8));
    }
}
```

- 在resources/META-INF/services目录下创建一个文件，文件名为org.pytorch.serve.servingsdk.ModelServerEndpoint（从 [serving-sdk](https://github.com/pytorch/serve/tree/master/serving-sdk) 继承的Java 类），文件内容为org.pytorch.serve.plugins.endpoint.S3DownloaderEndpoint（实现的Java接口类名）
- 在 build.gradle 编写配置文件和依赖项，并将 S3 文件整体打包为一个 S3download.jar 文件放在 Torch Serve 的 plugin 目录下。
- 使用配置文件 `plugins_path=<path-containing-plugin-jars">`或者命令行`torchserve --start --model-store <your-model-store-path> --plugins-path=<path-to-plugin-jars>`设置指向JAR文件的配置。
- 当插件配置好后我们可以初始化 Torch Serve，以及通过Manage API 从 S3 下载文件。
```
$ torchserve --start --model-store <your-model-store-path> --plugins-path=<path-to-plugin-jars>
$ curl "http://localhost:8080/s3download?bucket=file_bucket&key=file_key"
```


## 其他
### 日志与指标

1. logging

TorchServe目前提供访问日志（存储在access_log.log 文件中）和 Torch Serve日志（存储在ts_log.log中）。
```
#access log
2018-10-15 13:56:18,976 [INFO ] BackendWorker-9000 ACCESS_LOG - /127.0.0.1:64003 "POST /predictions/resnet-18 HTTP/1.1" 200 118

#torch serve log
#out info
2023-05-10T00:16:01,626 [INFO ] main org.pytorch.serve.ModelServer - Metrics API bind to: http://127.0.0.1:8082
#error
2023-05-10T00:16:02,117 [WARN ] pool-3-thread-1 org.pytorch.serve.metrics.MetricCollector - worker pid is not available yet.
```
如果您的模型是超轻量级的并且您想要高吞吐量，可以考虑启用异步日志记录。但如果 TorchServe 意外终止，日志输出可能会延迟，并且最近的日志可能会丢失。默认情况下禁用异步日志记录。要启用异步日志记录，请在中添加async_logging=true到config.properies。

2 .Metrics
TorchServe指标分为前端指标和后端指标。前端指标为系统指标，每一分钟自动收集一次。后端指标可以通过API访问。指标支持log和Prometheus形式，默认形式为log。在log形式下，前端指标存储在ts_metrics.log，后端指标存储在model_metrics.log。而在prometheus形式下所有的指标可以通过API访问(通过端口8082监听，且只能通过本地）
```
curl http://127.0.0.1:8082/metrics
```
[前后端指标具体项目](https://github.com/pytorch/serve/blob/master/docs/metrics.md)
*用户可以编写YAML文件自定义指标，并通过config.properties修改指标文件路径
### 备注

- 常见的错误代码及解决方案: [https://pytorch.org/serve/Troubleshooting.html](https://pytorch.org/serve/Troubleshooting.html)
- 用户可以自主修改torchserve默认参数，如监听端口，输出格式，API使用等。常用的配置方式按优先级分为环境变量，命令行参数(API)，配置文件(config.properies)。具体配置方法查看：[https://github.com/pytorch/serve/blob/master/docs/configuration.md](https://github.com/pytorch/serve/blob/master/docs/configuration.md)
### 
