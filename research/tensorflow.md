# Tensorflow
## 1、概述
TensorFlow Serving提供了一个开箱即用的模型推理环境，通常情况下，只要将训练好的模型下载到文件系统上，并开启TensorFlow Serving服务，即可通过REST API或者gRPC协议访问TensorFlow Serving，获取模型推理的结果。
使用docker部署一个简单的TensorFlow Serving步骤如下：

- 将训练好的模型放在文件系统的目录上。
- 启动docker容器并挂载模型存放路径

`docker run -t --rm -p 8501:8501 -v {location} \`
`-e MODEL_NAME={model_name} \`
`tensorflow/serving &`

- 通过REST API访问

`curl -d  {data} -X POST http://localhost:8501/v1/{location}:predict`
## 2、关键概念
### 2.1、Servables
Servable是对任何可处理服务请求的实体的抽象，它可以是任意的类型和实体，例如一个查找表、一个模型。其类定义如下所示：
```cpp
//定义在tensorflow_serving/core/servable_data.h
template <typename T>
class ServableData {
	...
private:
  ServableData() = delete;

  const ServableId id_;
  const Status status_;
  T data_;
}
```
Servable具有version，并且TensorFlow Serving可以同时处理多个不同version的特定模型，客户端即可以请求最新版的模型，也可以请求一个旧版本的模型。`ServableId`结构定义了Servable的name和version：
```cpp
//定义在tensorflow_serving/core/servable_id.h
struct ServableId {
//标识servable stream中的一个Servable
string name;
int64_t version;
...
}
```
servable stream是一系列不同version的Servable。
### 2.2、Loaders
Servable本身并不管理生命周期，由Loaders负责加载或卸载一个Servable。Loaders API支持独立于所涉及的特定学习算法、数据或产品用例的通用基础架构。Loader接口定义如下：
```cpp
//定义在tensorflow_serving/core/loader.h
class Loader {
public:
	virtual ~Loader() = default;
	//评估一个Servable将要消耗的资源，以确保Servable被安全的装载
	virtual Status EstimateResources(ResourceAllocation* estimate) const = 0;
	virtual Status Load() {
    	return errors::Unimplemented("Load isn't implemented.");
  	}
	struct Metadata {
    	ServableId servable_id;
  	};
	virtual Status LoadWithMetadata(const Metadata& metadata) { return Load(); }
	virtual void Unload() = 0;
	virtual AnyPtr servable() = 0;
}
```
一个Loader并不直接被`Manager`持有，LoaderHarness类负责持有Loader并通信，定义如下：
```cpp
//定义在tensorflow_serving/core/loader_harness.h
class LoaderHarness final {
public:
	...
private:
	...
	const ServableId id_;
  	const std::unique_ptr<Loader> loader_;
    const UniqueAnyPtr additional_state_;
  	const Options options_;
  	mutable mutex mu_;
	...
}
```
### 2.3、Sources
Sources是一个plugin模块，负责发现和提供servables。每一个Source提供0个或多个servable streams。对于每一个servable streams，Source为每一个version提供一个Loader实例。Source可以从任意的存储系统发现servables。
Source负责处理可以用来加载servables的数据，例如：

- 含有一个序列化的词映射的文件系统路径
- 指定机器学习模型加载的RPC
- 一个Loader

Source的抽象定义如下：
```cpp
//定义在tensorflow_serving/core/source.h
template <typename T>
class Source {
public:
  	virtual ~Source() = default;
	using AspiredVersionsCallback = std::function<void(
      const StringPiece servable_name, std::vector<ServableData<T>> versions)>;
	virtual void SetAspiredVersionsCallback(AspiredVersionsCallback callback) = 0;
};
```
SourceAdapter抽象类负责将具有一种类型的输入，转换成另一种类型的输出。例如，InputType=StoragePath，OutputType=unique_ptr<loader>，SourceAdapter可以将每一个存储路径转变成一个Loader。定义如下所示：
```cpp
// 定义在tensorflow_serving/core/source_adapter.h
template <typename InputType, typename OutputType>
class SourceAdapter : public TargetBase<InputType>, public Source<OutputType> {
public:
	~SourceAdapter() override = 0;
	void SetAspiredVersions(const StringPiece servable_name,
                          std::vector<ServableData<InputType>> versions) final;
	void SetAspiredVersionsCallback(
    	typename Source<OutputType>::AspiredVersionsCallback callback) final;
	//进行转换的方法
	virtual std::vector<ServableData<OutputType>> Adapt(
      	const StringPiece servable_name,
      	std::vector<ServableData<InputType>> versions) = 0;
	ServableData<OutputType> AdaptOneVersion(ServableData<InputType> input);
protected:
  	SourceAdapter() = default;

private:
  	typename Source<OutputType>::AspiredVersionsCallback outgoing_callback_;
  	Notification outgoing_callback_set_;
}

```
在实际使用一个Source时，通常将多个SourceAdapter连接在一起，进行多个类型转换，最后输出一个Loader。
### 2.4、Aspired Versions
Aspired versions表示需要被加载的servable的versions集合。Source会将该集合的version作为一个servable stream，当一个新的Aspired versions被提交给Manage后，先前的会被替换掉，Manage会卸载掉不在列表中的version。
### 2.5、Managers
Managers处理Servables的全部生命周期，包括：

- loading Servables
- serving Servables
- unloading Servables

Manager会监听Source的请求，并尽力满足加载一个Servable的请求(资源满足的情况下)。Manager的定义如下所示：
```cpp
class Manager {
public:
	virtual ~Manager() = default;
	// 获取所有的可用Servables的Id
	virtual std::vector<ServableId> ListAvailableServableIds() const = 0;
	// 获取所有当前可用的Servables
	template <typename T>
  	std::map<ServableId, ServableHandle<T>> GetAvailableServableHandles() const;
	// 根据请求获取一个servable
	template <typename T>
  	Status GetServableHandle(const ServableRequest& request,
                           ServableHandle<T>* const handle);
	 
private:
  	friend class ManagerWrapper;
	virtual Status GetUntypedServableHandle(
      	const ServableRequest& request,
      	std::unique_ptr<UntypedServableHandle>* untyped_handle) = 0;
	virtual std::map<ServableId, std::unique_ptr<UntypedServableHandle>>
  	GetAvailableUntypedServableHandles() const = 0;

```
### 2.6、架构
![serving_architecture.svg](https://cdn.nlark.com/yuque/0/2023/svg/35903190/1683620184305-5a86c901-9801-4005-9a50-823306ac90da.svg#clientId=u0f4af153-fb2a-4&from=ui&height=326&id=ue0950928&originHeight=150&originWidth=289&originalType=binary&ratio=1&rotation=0&showTitle=false&size=356119&status=done&style=none&taskId=uf4214beb-6d82-4473-abd8-a80d2210fc5&title=&width=629)

1. Source plugin创建一个特定版本的Loader。这个Loader包含所有加载一个servable所必须的元数据。
2. Source使用一个callback通知Manager一个Aspired Version。
3. Manager应用某个策略执行下一个活动，包括加载或卸载Servable。
4. 如果资源满足，Manager通知Loader加载一个新的version。
5. 客户端请求一个服务。
## 3、ModelServer
### 3.1、简述
ModelServer是Tensorflow Serving的实际例程。Tensorflow serving提供了一个标准的ModelServer，可以满足一般的需求。同时，可以使用Tensorflow serving组件构建一个标准的ModelServer，以发现新训练的模型。
### 3.2、构建一个标准的ModleServer

1. 训练模型并将模型导入到目录中。
2. 使用 ServerCore构建服务
```cpp
int main(int argc, char** argv) {
  ...
  /// 一些常用的配置选项
  ServerCore::Options options;
  options.model_server_config = model_server_config;
  options.servable_state_monitor_creator = &CreateServableStateMonitor;
  options.custom_model_config_loader = &LoadCustomModelConfig;

  ::google::protobuf::Any source_adapter_config;
  SavedModelBundleSourceAdapterConfig
      saved_model_bundle_source_adapter_config;
  source_adapter_config.PackFrom(saved_model_bundle_source_adapter_config);
  (*(*options.platform_config_map.mutable_platform_configs())
      [kTensorFlowModelPlatform].mutable_source_adapter_config()) =
      source_adapter_config;

  /// 根据配置文件创建一个ServerCore
  /// ServerCore内部包装了一个AspiredVersionsManager
  /// SavedModelBundle提供了加载模型和推理的方法
  /// SavedModelBundleSourceAdapter将一个storage path转变成一个Loader<SavedModelBundle>
  /// ServerCore内部处理流程如下：
  /// 	* 实例化一个FileSystemStoragePathSource，检测模型
  /// 	* 实例化一个SourceAdapter，以生成Loader<SavedModelBundle>
  /// 	* 实例化一个Manager实现：AspiredVersionsManager。
  std::unique_ptr<ServerCore> core;
  TF_CHECK_OK(ServerCore::Create(options, &core));
  RunServer(port, std::move(core));

  return 0;
}
```

3. 批量处理：使用**SessionBundleConfig**配置
## 4、自定义新类型
### 4.1、创建一个新类型的servable

1. 首先需要创建 Loader和 SourceAdapter。Loader可以从任何地方加载数据，继承Loader类和sourceAdapt类自定义一个loader。以tensorflow serving实现的**std::map<string, string>**为例，如下所示：
```cpp
// 定义在tensorflow_serving/servables/hashmap/hashmap_source_adapter.h
class HashmapSourceAdapter final
    : public SimpleLoaderSourceAdapter<StoragePath,
                                       std::unordered_map<string, string>> {
 public:
  explicit HashmapSourceAdapter(const HashmapSourceAdapterConfig& config);
  ~HashmapSourceAdapter() override;

 private:
  TF_DISALLOW_COPY_AND_ASSIGN(HashmapSourceAdapter);
};

// 定义在tensorflow_serving/servables/hashmap/hashmap_source_adapter.cc
Status LoadHashmapFromFile(const string& path,
                           const HashmapSourceAdapterConfig::Format& format,
                           std::unique_ptr<Hashmap>* hashmap) {
  hashmap->reset(new Hashmap);
  switch (format) {
    case HashmapSourceAdapterConfig::SIMPLE_CSV: {
      std::unique_ptr<RandomAccessFile> file;
      TF_RETURN_IF_ERROR(Env::Default()->NewRandomAccessFile(path, &file));
      const size_t kBufferSizeBytes = 262144;
      io::InputBuffer in(file.get(), kBufferSizeBytes);
      string line;
      while (in.ReadLine(&line).ok()) {
        std::vector<string> cols = str_util::Split(line, ',');
        if (cols.size() != 2) {
          return errors::InvalidArgument("Unexpected format.");
        }
        const string& key = cols[0];
        const string& value = cols[1];
        (*hashmap)->insert({key, value});
      }
      break;
    }
    default:
      return errors::InvalidArgument("Unrecognized format enum value: ",
                                     format);
  }
  return Status();
}

```

2. 让manager管理自定义的Servable
```cpp
// 创建一个Manager
std::unique_ptr<AspiredVersionsManager> manager = ...;
// 创建自定义Servable的Source adapter，并注册进manager
auto your_adapter = new YourServableSourceAdapter(...);
ConnectSourceToTarget(your_adapter, manager.get());
// 创建一个路径源并关联到自定义的Source adapter
ConnectSourceToTarget(path_source.get(), your_adapter.get());
```

3. 访问已加载的Servable对象
```cpp
auto handle_request = serving::ServableRequest::Latest("default");
ServableHandle<YourServable*> servable;
Status status = manager->GetServableHandle(handle_request, &servable);
if (!status.ok()) {
  LOG(INFO) << "Zero versions of 'default' servable have been loaded so far";
  return;
}
// Use the servable.
(*servable)->SomeYourServableMethod();
```
### 4.2、创建新的模块发现新Servable路径
以官方实现的**FileSystemStoragePathSource**为例

1. **FileSystemStoragePathSource**继承`Source<StoragePath>`，**SetAspiredVersionsCallback()**方法提供了希望加载特定可服务版本的方法。
2. 周期性的轮询文件系统以发现Servable。



# TensorFlow Serving 分析
![时序图.png](https://cdn.nlark.com/yuque/0/2023/png/35903190/1684742441115-5aed3d83-9139-4691-a6c0-8606b16faa78.png#averageHue=%232b2b2b&clientId=u9962fa02-0f04-4&from=ui&id=u77e48cd8&originHeight=710&originWidth=890&originalType=binary&ratio=1&rotation=0&showTitle=false&size=41534&status=done&style=none&taskId=u4ffba030-2109-41e6-8c7d-bd42dbfc127&title=)
## 1、从 main 函数开始
![REST.png](https://cdn.nlark.com/yuque/0/2023/png/35903190/1683955286493-537efd51-963c-4d62-aac3-7c40fd493233.png#averageHue=%23f6f6f6&clientId=u92883691-08af-4&from=ui&id=u1f30b2a3&originHeight=347&originWidth=715&originalType=binary&ratio=1&rotation=0&showTitle=false&size=34879&status=done&style=none&taskId=u22779ec7-2a80-4a31-a541-9a182416cad&title=)
main 函数首先解析命令行参数，并配置 Options 类。Options 类可配置参数如下所示：

| 参数名 | 含义 |
| --- | --- |
| port | 监听grpc的端口 |
| grpc_socket_path | 监听grpc socket 路径 |
| rest_api_port | 监听REST的端口 |
| rest_api_num_threads | 处理HTTP请求的线程数 |
| rest_api_timeout_in_ms | 处理请求超时配置 |
| rest_api_enable_cors_support | 允许回应中的CORS头 |
| enable_batching | 允许批量处理 |
| allow_version_labels_for_unavailable_models | 允许给不可用的模型一个未使用的version标签 |
| batching_parameters_file | 从文件中读取 |
| enable_per_model_batching_parameters | 允许模型从SavedModel所在目录的`batching_params.pbtxt`文件中指定特定的batching params |
| model_config_file | 从文件中读取serving model |
| model_config_file_poll_wait_seconds | 从文件中轮询配置文件的时间间隔 |
| model_name | 模型名字 |
| model_base_path | 模型路径 |
| num_load_threads | 加载model servables的线程池数 |
| num_unload_threads | 卸载的线程池数 |
| max_num_load_retries | 加载一个model的最大尝试次数 |
| load_retry_interval_micros | 每次尝试加载的时间间隔 |
| file_system_poll_wait_seconds | 轮询文件系统的时间间隔 |
| flush_filesystem_caches | 刷新缓存 |
| tensorflow_session_parallelism | 运行tensorflow session的线程数 |
| tensorflow_session_config_file | tensorflow session的配置文件 |
| tensorflow_intra_op_parallelism | 并行执行独立操作的线程数 |
| tensorflow_inter_op_parallelism | 控制同时执行操作数的数量 |
| use_alts_credentials | 使用Google ALTS认证 |
| ssl_config_file | ssl配置文件 |
| platform_config_file | 平台配置文件 |
| per_process_gpu_memory_fraction | 进程占用gpu份额 |
| saved_model_tags | meta graph def tags |
| grpc_channel_arguments | grpc参数 |
| grpc_max_threads | grpc最大线程数 |
| enable_model_warmup | model warmup，减少模型第一次请求的时间 |
| num_request_iterations_for_warmup | 在warmup期间，一个请求的迭代次数 |
| version | Display version |
| monitoring_config_file | 配置文件 |
| remove_unused_fields_from_bundle_metagraph | 移除未使用的fields |
| prefer_tflite_model | 优先使用`model.tflite`文件 |
| num_tflite_pools | TfLiteSession中一个interpreter pool中的TFLite interpreters数量 |
| num_tflite_interpreters_per_pool | TfLiteSession中一个interpreter pool中的TFLite interpreters数量 |
| enable_signature_method_name_check | SignatureDef的方法名检查 |
| xla_cpu_compilation_enabled | 允许XLA:CPU JIT |
| enable_profiler | 允许profiler服务 |
| thread_pool_factory_config_file | 配置文件 |

接着根据 Options 类配置 Server ，调用 WaitForTermination() 方法启动服务。WaitForTermination() 内部会判断是否启用 REST 和 gRPC 服务，例如如果启用了 REST 服务，最终会调用 EvHTTPServer::WaitForTermination() 方法循环等待请求并分发处理请求。
![handler.png](https://cdn.nlark.com/yuque/0/2023/png/35903190/1683961005079-7d99ad53-649a-4e50-9eb6-a4654401f276.png#averageHue=%23f9f9f9&clientId=u92883691-08af-4&from=ui&id=u4302b70d&originHeight=539&originWidth=3046&originalType=binary&ratio=1&rotation=0&showTitle=false&size=88606&status=done&style=none&taskId=u4e44ba88-532f-43bd-b10d-4c3c78260ad&title=)
HttpRestApiHandler::ProcessRequest() 方法处理 REST 传入的请求，该方法会根据路径中的请求参数决定实际调用哪个函数处理。例如，对于 predict 任务，会调用 TensorflowPredictor::Predict() 方法。该方法获取到 modelspec ，并调用 TensorflowPredictor::PredictWithModelSpec() 方法。该方法首先根据 ServerCore获取到一个 ServableHandle 并执行最终的预测。
## 2、ServableHandle
```cpp
class UntypedServableHandle {
 public:
  virtual ~UntypedServableHandle() = default;
  virtual const ServableId& id() const = 0;
  virtual AnyPtr servable() = 0;
};

template <typename T>
class ServableHandle {
public:
	......
    const ServableId& id() const { return untyped_handle_->id(); }
    T& operator*() const { return *get(); }
  	T* operator->() const { return get(); }
  	T* get() const { return servable_; }
  	operator bool() const { return get() != nullptr; }
private:
	friend class Manager;
	......
    std::unique_ptr<UntypedServableHandle> untyped_handle_;
  	T* servable_ = nullptr;
}

class SharedPtrHandle final : public UntypedServableHandle {
public:
	......
  	AnyPtr servable() override { return loader_->servable(); }
  	const ServableId& id() const override { return id_; }
private:
  	const ServableId id_;
  	std::shared_ptr<Loader> loader_;
};
```

- UntypedServableHandle 抽象类定义了 id() 和 servable() 抽象方法，其实现类 SharedPtrHandle 通过 id() 返回一个 ServableId 对象；通过 servable() 调用了 Loader 类的 servable()，其返回一个 AnyPtr 类型的对象（实质将 void * 封装了一下）。
- ServableHandle 类主要重载了指针运算符，返回一个模板对象。

总结：ServableHandle 类主要返回一个 servable 对象。
## 3、ServerCore
### 3.1、ServerCore::Options
ServerCore::Options 类定义了创建 ServerCore 的所有配置选项，其配置项如下所示：

| 参数名 | 含义 |
| --- | --- |
| model_server_config | 指定 model 如何加载 |
| model_config_list_root_dir | model_server_config中指定的相对路径会以该路径为前缀 |
| aspired_version_policy | manager使用的策略 |
| num_load_threads | 加载模型的线程数 |
| num_initial_load_threads | 启动时使用的线程数 |
| num_unload_threads | 卸载模型使用的线程数 |
| total_model_memory_limit_bytes | 模型大小限制 |
| max_num_load_retries | 最大重新尝试的次数 |
| load_retry_interval_micros | 重试的间隔 |
| file_system_poll_wait_seconds | 文件系统轮询的间隔 |
| flush_filesystem_caches | 缓存刷新 |
| platform_config_map | 支持平台 |
| servable_state_monitor_creator | 创建 ServableStateMonitor 的函数 |
| custom_model_config_loader | 实例化和关联 sources 和 source adapters 到manager 的函数 |
| allow_version_labels | 是否禁止 'version_label' 字段 |
| enable_reload_servables_with_error | 允许错误后继续进行 |
| servable_versions_always_present | 如果设置为 true，会在指定目录上没有模型时无法启动服务 |
| server_request_logger | 日志 |
| server_request_logger_updater | 更新 server_request_logger 的函数 |
| pre_load_hook | 回调函数，在 servable 被加载时调用 |
| allow_version_labels_for_unavailable_models | 是否允许给不可用的模型一个未使用的 version |
| force_allow_any_version_labels_for_unavailable_models | 强迫给不可用的模型一个未使用的 version |
| predict_response_tensor_serialization_option | 指定如何序列化 tensor |
| storage_path_prefix | 文件系统路径前缀 |
| enable_cors_support | CORS |

### 3.2、实例化 ServerCore
 ![servercore.png](https://cdn.nlark.com/yuque/0/2023/png/35903190/1683972914973-e85879b5-b961-4a19-a428-11c484a2a829.png#averageHue=%23f6f6f6&clientId=ucae1c752-9fcb-4&from=ui&id=u87a534ba&originHeight=251&originWidth=736&originalType=binary&ratio=1&rotation=0&showTitle=false&size=22282&status=done&style=none&taskId=ub6f471e2-0ad5-4d49-b42a-a05ce60ab01&title=)

- main() 函数中调用 Server::BuildAndStart() 方法初始化服务。
- 根据 Server::Options 实例初始化 ServerCore::Options 。
- 根据 ServerCore::Options 实例化 ServerCore 。
### 3.3、ServerCore 创建 AspiredVersionsManager
```cpp
class ServerCore : public Manager {
public:
    // 回调函数，在 servable 加载之前调用
 	using PreLoadHook = AspiredVersionsManager::PreLoadHook;
	// 一个函数负责实例化和关联 source/source adapters 到 manager
	// 通过传递一个 config(any)
	// 目前不支持？
	using CustomModelConfigLoader = std::function<Status(
      const ::google::protobuf::Any& any, EventBus<ServableState>* event_bus,
      UniquePtrWithDeps<AspiredVersionsManager>* manager)>;
......
private:
......
Options options_;
std::map<string, int> platform_to_router_port_;
std::shared_ptr<EventBus<ServableState>> servable_event_bus_;
std::shared_ptr<ServableStateMonitor> servable_state_monitor_;
UniquePtrWithDeps<AspiredVersionsManager> manager_;
......
}
```
ServerCore 类定义了一系列处理 Servable 的函数，并且内嵌了一个 AspiredVersionsManager 实例，其负责管理 Servable 的全部生命周期。
![AspiredVersionsManager.png](https://cdn.nlark.com/yuque/0/2023/png/35903190/1683977068105-602e9ad6-15ea-4dcf-bba0-7cb2311552c4.png#averageHue=%23f5f5f5&clientId=ucae1c752-9fcb-4&from=ui&id=ubb23c6c3&originHeight=347&originWidth=638&originalType=binary&ratio=1&rotation=0&showTitle=false&size=30872&status=done&style=none&taskId=ue7cb5594-f9cc-4dc3-9119-80e5aadf287&title=)
使用 ServerCore::Create 创建 ServerCore 时会创建一个 AspiredVersionsManager 实例。
### 3.4、AspiredVersionPolicy
AspiredVersionPolicy 定义了在 servable stream 中转换 servable versions的策略。
```cpp
class AspiredVersionPolicy {
public:
	enum class Action : int {
    	/// load 一个 servable
    	kLoad,
    	/// unload 一个 servable.
    	kUnload,
  	};
	 struct ServableAction final {
    	Action action;
    	ServableId id;
    	string DebugString() const {
      	return strings::StrCat("{ action: ", static_cast<int>(action),
                             " id: ", id.DebugString(), " }");
    	}
  	};
	/// 从所有的 servable stream 中取出一个 version 的 Action 
	virtual absl::optional<ServableAction> GetNextAction(
      const std::vector<AspiredServableStateSnapshot>& all_versions) const = 0;
	/// 返回最高版本号
	static absl::optional<ServableId> GetHighestAspiredNewServableId(
      const std::vector<AspiredServableStateSnapshot>& all_versions);
}
```

1. ServerCore 使用了 AvailabilityPreservingPolicy ，实现了 AspiredVersionPolicy 。其中 GetNextAction() 方法的处理流程如下：
2. 检查出所有的 non-aspired versions。
3. 如果没有 aspired version， 或者至少有一个 aspired version 在 ready 状态， 或者 non-aspired versions 数量大于2，则 unload 最低版本号。
4. 如果有最新的 version， 则 load。
5. 否则返回一个空的 action。
## 4、Manage
![manager.png](https://cdn.nlark.com/yuque/0/2023/png/35903190/1684046719918-feac6140-581f-4196-ac6c-ad97dffbbce3.png#averageHue=%23838383&clientId=uef6183c2-9559-4&from=ui&id=u66d7a35f&originHeight=1304&originWidth=1512&originalType=binary&ratio=1.5&rotation=0&showTitle=false&size=272151&status=done&style=none&taskId=u80b1b07d-0650-491c-b7f6-9dc699b6aec&title=)
### 5.1、加载 ServableHandle
![handler.png](https://cdn.nlark.com/yuque/0/2023/png/35903190/1684052457823-62cef177-fc01-441a-8a9d-7afae96dd5a7.png#averageHue=%23eeeeee&clientId=uef6183c2-9559-4&from=ui&id=u64f6568a&originHeight=155&originWidth=1114&originalType=binary&ratio=1.5&rotation=0&showTitle=false&size=23216&status=done&style=none&taskId=ueec297b6-1666-4cab-a83c-859ffbcc576&title=)
结构体 ServableRequest 封装了 servable 的 name 和 version，用以指示特定的 servable 对象。Manager 类的 public 方法 GetServableHandle() 和 GetAvailableServableHandles() 实际上分别调用 private 方法 GetUntypedServableHandle() 和 GetAvailableUntypedServableHandles() 。
在 BasicManager 的实现中，进一步委托给 ServingMap 类实现，其定义如下所示：
```cpp
class ServingMap{
public:
  std::vector<ServableId> ListAvailableServableIds() const;
  Status GetUntypedServableHandle(
        const ServableRequest& request,
        std::unique_ptr<UntypedServableHandle>* untyped_handle);
  std::map<ServableId, std::unique_ptr<UntypedServableHandle>>
    GetAvailableUntypedServableHandles() const;
  void Update(const ManagedMap& managed_map);
private:
  struct EqRequest;
  struct HashRequest;
  using HandlesMap =
        std::unordered_multimap<ServableRequest,
                                std::shared_ptr<const LoaderHarness>,
                                HashRequest, EqRequest>;
    FastReadDynamicPtr<HandlesMap> handles_map_;
}
```
ServingMap::GetUntypedServableHandle() 方法中在 handles_map_ 根据 ServableRequest 参数查找出相应的 LoaderHarness ，接着构造一个 UntypedServableHandle 。ServingMap::GetAvailableUntypedServableHandles()  方法遍历 handles_map_ 跳过所有的 auto-versioned 。
### 5.2、执行 load 和 unload
![loadOrUnload.png](https://cdn.nlark.com/yuque/0/2023/png/35903190/1684064526655-8e8ec6eb-6537-4bf6-af2c-5ffbbc55234c.png#averageHue=%23f5f5f5&clientId=ucc55a534-40a9-4&from=ui&id=u4031f96b&originHeight=635&originWidth=720&originalType=binary&ratio=1.5&rotation=0&showTitle=false&size=60263&status=done&style=none&taskId=u1bcfeb0f-c202-4091-bfd7-6f9e619f1d4&title=)

1. Executor：BasicManager 使用两种执行器：一种是 InlineExecutor，在当前线程执行任务；一种是 ThreadPoolExecutor ，会使用一个线程池执行任务。
2. BasicManager::LoadServable() 和 BasicManager::UnloadServable() 方法内部调用 BasicManager::LoadOrUnloadServable() 方法。其内部根据一个枚举量决定 load 或 unload ，并启用 Executor 执行任务，传入一个函数表达式，调用 BasicManager::HandleLoadOrUnloadRequest() 方法。
3. BasicManager::HandleLoadOrUnloadRequest() 方法首先判断是否支持 load 或 unload，接着调用 BasicManager::ExecuteLoadOrUnload()方法，
4. BasicManager::ExecuteLoadOrUnload() 方法根据请求类型分别执行 BasicManager::ExecuteUnload() 和 BasicManager::ExecuteLoad 方法。
5. BasicManager::ExecuteUnload() 和 BasicManager::ExecuteLoad 方法将事件发布到 EventBus 上。
### 5.3、AspiredVersionsManager 实现

1. 大部分的函数接口调用 BasicManager 实现。 
2. AspiredVersionsManager::ProcessAspiredVersionsRequest() 方法将当前的 Aspired Versions 替换掉。
3. AspiredVersionsManager::GetNextAction() 方法从 servable stream 流中选出下一个 Action 。
4. AspiredVersionsManager::PerformAction() 方法执行一个 Action ，主要包括 load 和 unload，分别使用BasicManager::LoadServable() 和 BasicManager::UnloadServable() 方法。
5. AspiredVersionsManager::FlushServables() 移除所有无用的 servable 。
6. 如果配置了 manage_state_interval_micros ，AspiredVersionsManager 构造函数会启用一个周期函数，执行 FlushServables() ，HandlePendingAspiredVersionsRequests() ，InvokePolicyAndExecuteAction()。 HandlePendingAspiredVersionsRequests() 调用 ProcessAspiredVersionsRequest() ;InvokePolicyAndExecuteAction() 调用 GetNextAction() 和 PerformAction() 。
7. AspiredVersionsManager 继承了 TargetBase() 。TargetBase 类实现了 SetAspiredVersions() 方法 ，其实现如下：
```cpp
/// Observer 类是对 std::function 的封装
std::unique_ptr<Observer<const StringPiece, std::vector<ServableData<T>>>>
      observer_;

template <typename T>
typename Source<T>::AspiredVersionsCallback
TargetBase<T>::GetAspiredVersionsCallback() {
  mutex_lock l(*mu_);
  if (detached_->HasBeenNotified()) {
    // We're detached. Return a no-op callback.
    return [](const StringPiece, std::vector<ServableData<T>>) {};
  }
  return observer_->Notifier();
}  
```

8. TargetBase 构造函数构建了一个 Observer，实现了一个匿名函数，内部调用了 SetAspiredVersions() 方法。AspiredVersionsManager 的实现中，该方法调用 EnqueueAspiredVersionsRequest() 方法，将 Aspired Versions 加入到等待队列中。会在 HandlePendingAspiredVersionsRequests() 方法中处理。
```cpp
template <typename T>
TargetBase<T>::TargetBase() : mu_(new mutex), detached_(new Notification) {
  ......
  [mu, detached, this](const StringPiece servable_name,
                           std::vector<ServableData<T>> versions) {
        mutex_lock l(*mu);
        if (detached->HasBeenNotified()) {
          // We're detached. Perform a no-op.
          return;
        }
        this->SetAspiredVersions(servable_name, std::move(versions));
    }));
  ......
}
```

9. ConnectSourceToTarget() 方法将一个 Source 连接到一个 Target，其实现如下：
```cpp
template <typename T>
void ConnectSourceToTarget(Source<T>* source, Target<T>* target) {
  source->SetAspiredVersionsCallback(target->GetAspiredVersionsCallback());
}
```
## 5、Source
![source_class.png](https://cdn.nlark.com/yuque/0/2023/png/35903190/1684120869204-940d1921-ef9b-4907-846d-c1b9ed944698.png#averageHue=%23454545&clientId=ucae1c752-9fcb-4&from=ui&id=u42d81220&originHeight=711&originWidth=1413&originalType=binary&ratio=1&rotation=0&showTitle=false&size=87809&status=done&style=none&taskId=uaf646dd6-b34f-48aa-9a95-f56e752bf87&title=)
### 5.1、SourceRouter 模块的作用
![source_route.png](https://cdn.nlark.com/yuque/0/2023/png/35903190/1684133798219-09eff4af-981f-4261-812d-91ec03f54684.png#averageHue=%23f6f6f6&clientId=ucae1c752-9fcb-4&from=ui&id=udfcd2548&originHeight=251&originWidth=806&originalType=binary&ratio=1&rotation=0&showTitle=false&size=23788&status=done&style=none&taskId=u2606316e-58fc-4231-8aa4-25549fdc721&title=)
SourceRouter 会将一个 aspired-version 请求调用划分成多个 SourceAdapter 类型（实际实现是一个等量输出的 SourceAdapter ，Adapt() 方法直接将输入返回）的输出。例如，SourceRouter 插入到文件系统监视源 Source<StoragePath>, 和多个 SourceAdapt 之间，基于某种原则，将对应的路径路由到恰当的 SourceAdapt 上。
tensorflow serving 实现了两种 SourceRouter ：StaticSourceRouter 和 DynamicSourceRouter 。StaticSourceRouter 使用一个字符串数组，route() 方法返回匹配的字符串数组下标。DynamicSourceRouter 使用一个 map<string, int>，并提供了更新这个 map 的方法。 
### 5.2、SourceAdapt 类的作用
Adapt() 方法接受一个 servable name 和 一种类型 servable 数组，输出另一种类型的 servable 数组。SourceAdapt::SetAspiredVersions()方法实现如下：
```cpp
template <typename InputType, typename OutputType>
void SourceAdapter<InputType, OutputType>::SetAspiredVersions(
    const StringPiece servable_name,
    std::vector<ServableData<InputType>> versions) {
  outgoing_callback_set_.WaitForNotification();
  outgoing_callback_(servable_name, Adapt(servable_name, std::move(versions)));
}
```
多个 SourceAdapt 连接在一起，会依次调用 SetAspiredVersions() 方法，产生一个最终的输出。如下图所示：
![source_adapt.png](https://cdn.nlark.com/yuque/0/2023/png/35903190/1684135488845-ee018cb2-52d9-4c3d-a82a-6cf55177228c.png#averageHue=%23f4f4f4&clientId=ucae1c752-9fcb-4&from=ui&id=u8bf94dc7&originHeight=443&originWidth=342&originalType=binary&ratio=1&rotation=0&showTitle=false&size=25030&status=done&style=none&taskId=ude1e97ad-0a9a-432d-8b2f-2165b470c52&title=)
## 6、loader
![loader_class.png](https://cdn.nlark.com/yuque/0/2023/png/35903190/1684140262450-61c125be-f0be-443a-be0e-1afc3a5e09cc.png#averageHue=%23939393&clientId=ucae1c752-9fcb-4&from=ui&id=ud0ea3b45&originHeight=536&originWidth=451&originalType=binary&ratio=1&rotation=0&showTitle=false&size=33199&status=done&style=none&taskId=u2f90d866-4e08-4fdc-9f92-8905c69d23c&title=)
SimpleLoaderSourceAdapter 类将一个 DataType 转变成一个 SimpleLoader。
## 7、文件系统
### 7.1、ServerCore 初始化 Source
![server_core_init.png](https://cdn.nlark.com/yuque/0/2023/png/35903190/1684720921817-fa18409d-5eeb-4102-ae27-4b87246fa4ef.png#averageHue=%23f7f7f7&clientId=u6f8ec33b-89a3-4&from=ui&id=u46db9c5d&originHeight=443&originWidth=1506&originalType=binary&ratio=1&rotation=0&showTitle=false&size=69691&status=done&style=none&taskId=ua90e65d6-1e99-43b4-bb92-a0f9a33c84d&title=)
创建 ServerCore 时，执行如下的流程：

1. 调用 ReloadConfig(ModelServerConfig&) 方法。执行2 。
2. 调用 AddModelsViaModelConfigList() 或者 AddModelsViaCustomModelConfig() 方法。执行3、4 。
3. 调用 CreateStoragePathRoutes() 方法。该方法会根据模型对应的平台构建一个 Routes，每个输出端口对应一个平台。
4. 判断是否是第一次加载配置，若是的话，执行 5、6、7、8 ；若否，执行 9 。
5. 调用 CreateAdapters()。该方法会创建不同平台的 SourceAdapter<StoragePath, std::unique_ptr<Loader>> 类型。
```cpp
  Status ServerCore::CreateAdapter(
    const string& model_platform,
    std::unique_ptr<StoragePathSourceAdapter>* adapter) {
      // 根据平台找到对应的配置。
      auto config_it =
      options_.platform_config_map.platform_configs().find(model_platform);
      // 找到source_adapter的配置。
      const ::google::protobuf::Any& adapter_config =
      config_it->second.source_adapter_config();
      // 创建adapt，例如 SavedModelBundleSourceAdapter。
      StoragePathSourceAdapterRegistry::CreateFromAny(adapter_config, adapter);
}
```

6. CreateRouter() 方法将 Router 的出口通过 ConnectSourceToTarget() 方法关联到对应的 adapt。
7. CreateStoragePathSource() 方法关联一个 FileSystemStoragePathSource 到 6 中创建的 router。
8. ConnectAdaptersToManagerAndAwaitModelLoads() 方法将 adapts 关联到 manager。
9. 根据新的配置文件重新配置。

如果配置了 model_config_file_poll_wait_seconds 或者 model_config_file ，启动 Server 时会创建一个周期函数，周期性的执行 ReloadConfig() 。
### 7.2、FileSystemStoragePathSource

1. 初始化：Create() 方法创建一个 FileSystemStoragePathSource。只更新的配置。
2. ConnectSourceToTarget() ： 调用该方法会调用 FileSystemStoragePathSource::SetAspiredVersionsCallback() , 其会根据是否配置轮询文件系统启用一个线程，周期性的执行 PollFileSystemAndInvokeCallback() 方法。
3. PollFileSystemAndInvokeCallback() ：访问文件系统上的文件路径（使用 Env::Default() ，该方法在 tensorflow 库中实现），并执行回调函数。
### 7.3、加载 SavedModelBundle
SavedModelBundleSourceAdapter 根据一个 StoragePath 创建一个 SimpleLoader<SavedModelBundle> ，实际通过 Convert() 方法实现。Convert() 首先初始化一个创建器，该创建器如下：
```cpp
[bundle_factory, path](const Loader::Metadata& metadata,
                                  std::unique_ptr<SavedModelBundle>* bundle) {
      TF_RETURN_IF_ERROR(bundle_factory->CreateSavedModelBundleWithMetadata(
          metadata, path, bundle));
      MaybePublishMLMDStreamz(path, metadata.servable_id.name,
                              metadata.servable_id.version);
      if (bundle_factory->config().enable_model_warmup()) {
        return RunSavedModelWarmup(
            bundle_factory->config().model_warmup_options(),
            GetRunOptions(bundle_factory->config()), path, bundle->get());
      }
      return OkStatus();
    };
```
创建器内部使用 CreateSavedModelBundleWithMetadata() 方法，该方法初始化一个 SavedModelBundle 实例，最终调用 LoadTfLiteModel() 方法。LoadTfLiteModel() 通过 Env::Default()->NewRandomAccessFile 获取到一个文件句柄，并读取文件。
SimpleLoader 中的 load() 方法会调用该创建器加载 SavedModelBundle 。
### 7.4、tensorflow 中的文件系统 plugin

- tensorflow serving 调用 tensorflow 中的文件系统实现处理文件 。
- tensorflow 实现的一些插件： [https://github.com/tensorflow/io/blob/master/tensorflow_io/core/filesystems/](https://github.com/tensorflow/io/blob/master/tensorflow_io/core/filesystems/)
- 关于 filesystem plugin 的 rfc 文档说明： [https://github.com/tensorflow/community/blob/master/rfcs/20190506-filesystem-plugin-modular-tensorflow.md](https://github.com/tensorflow/community/blob/master/rfcs/20190506-filesystem-plugin-modular-tensorflow.md)
- filesystem plugin 接口定义： [https://github.com/tensorflow/tensorflow/blob/master/tensorflow/c/experimental/filesystem/filesystem_interface.h](https://github.com/tensorflow/tensorflow/blob/master/tensorflow/c/experimental/filesystem/filesystem_interface.h)

定制一个文件系统 plugin 的一般步骤如下：

1. 实现 filesystem_interface.h 中定义的接口。
2. 编译成动态库文件。
3. 使用 tensorflow::RegisterFilesystemPlugin 注册动态库文件。

tensorflow 有两种注册方法，一种可能会放弃使用。调用链如下：

- TF_RegisterFilesystemPlugin -> tensorflow::RegisterFilesystemPlugin 。 python 中提供方法 register_filesystem_plugin 会调用 TF_RegisterFilesystemPlugin 。
- TF_LoadLibrary 方法会加载动态库，实现的插件可以通过宏 REGISTER_FILE_SYSTEM 注册。python 方法 load_file_system_library 会调用 TF_LoadLibrary 。
## 8、一些配置详解
--model_config_file
使用样例：`--model_config_file=/models/models.config`
传入的配置文件路径最终会被解析成 ModelServerConfig 类型，其定义为 protobuf ，如下：
```cpp
message ModelServerConfig {
  oneof config {
    ModelConfigList model_config_list = 1;
    google.protobuf.Any custom_model_config = 2;
  }
}
```
model_config_file 使用样例：   
```json
model_config_list {
  config {
  name: 'my_first_model'
  base_path: '/tmp/my_first_model/'
  model_platform: 'tensorflow'
}
config {
  name: 'my_second_model'
  base_path: '/tmp/my_second_model/'
  model_platform: 'tensorflow'
}

```
ModelConfigList 同样被定义为 protobuf，如下：
```cpp
message ModelConfigList {
  repeated ModelConfig config = 1;
}

message ModelConfig {
    string name = 1;
    // 指定一个模型的路径，例如 /foo/bar/my_model/123，
    // 其中 /foo/bar/my_model/ 是基路径，123是模型的版本。
    string base_path = 2;
    ModelType model_type = 3 [deprecated = true];
    string model_platform = 4;
    reserved 5, 9;
    FileSystemStoragePathSourceConfig.ServableVersionPolicy model_version_policy = 7;
    map<string, int64> version_labels = 8;
    LoggingConfig logging_config = 6;
}
```
model_version_policy 指定一个版本选择策略，使用样例：
```cpp
model_version_policy {
  specific {
    versions: 42
  }

```
其定义为：
```cpp
message ServableVersionPolicy {
    message Latest {
      uint32 num_versions = 1;
    }
    message All {}
    message Specific {
      repeated int64 versions = 1;
    }
    oneof policy_choice {
      Latest latest = 100;
      All all = 101;
      Specific specific = 102;
    }
}
```
其他 protobuf 定义的配置： [https://github.com/tensorflow/serving/tree/master/tensorflow_serving/config](https://github.com/tensorflow/serving/tree/master/tensorflow_serving/config)
## 9、API
> 关于 API 的 protobuf 文件定义在 [https://github.com/tensorflow/serving/tree/master/tensorflow_serving/apis](https://github.com/tensorflow/serving/tree/master/tensorflow_serving/apis)

### RESTful API 

1. 错误响应：
```cpp
{
  "error": <error message string>
}
```

2. 模型状态 api
```cpp
GET http://host:port/v1/models/${MODEL_NAME}[/versions/${VERSION}|/labels/${LABEL}]
```
如果成功，返回一个 GetModelStatusResponse protobuf 的 json 表示

3. 模型元数据 api
```cpp
GET http://host:port/v1/models/${MODEL_NAME}[/versions/${VERSION}|/labels/${LABEL}]/metadata
```
如果成功，返回一个 GetModelMetadataResponse protobuf 的 json 表示

4. 分类和回归 api
```cpp
POST http://host:port/v1/models/${MODEL_NAME}[/versions/${VERSION}|/labels/${LABEL}]:(classify|regress)
```
请求格式
```cpp
{
  // Optional: serving signature to use.
  // If unspecifed default serving signature is used.
  "signature_name": <string>,

  // Optional: Common context shared by all examples.
  // Features that appear here MUST NOT appear in examples (below).
  "context": {
    "<feature_name3>": <value>|<list>
    "<feature_name4>": <value>|<list>
  },

  // List of Example objects
  "examples": [
    {
      // Example 1
      "<feature_name1>": <value>|<list>,
      "<feature_name2>": <value>|<list>,
      ...
    },
    {
      // Example 2
      "<feature_name1>": <value>|<list>,
      "<feature_name2>": <value>|<list>,
      ...
    }
    ...
  ]
}
```
响应格式
```cpp
{
  "result": [
    // List of class label/score pairs for first Example (in request)
    [ [<label1>, <score1>], [<label2>, <score2>], ... ],

    // List of class label/score pairs for next Example (in request)
    [ [<label1>, <score1>], [<label2>, <score2>], ... ],
    ...
  ]
}

{
  // One regression value for each example in the request in the same order.
  "result": [ <value1>, <value2>, <value3>, ...]
}
```

5. 预测 api
```cpp
POST http://host:port/v1/models/${MODEL_NAME}[/versions/${VERSION}|/labels/${LABEL}]:predict
```
请求格式
```cpp
{
  // (Optional) Serving signature to use.
  // If unspecifed default serving signature is used.
  "signature_name": <string>,

  // Input Tensors in row ("instances") or columnar ("inputs") format.
  // A request can have either of them but NOT both.
  "instances": <value>|<(nested)list>|<list-of-objects>
  "inputs": <value>|<(nested)list>|<object>
}
```
响应格式
```cpp
// 行格式
{
  "predictions": <value>|<(nested)list>|<list-of-objects>
}
// 列格式
{
  "outputs": <value>|<(nested)list>|<object>
}
```

6. 二进制格式
```cpp
{ "b64": <base64 encoded string> }
```
# Tensorflow 数据集
## **TensorFlow Datasets**
### 简介
TFDS 提供了一个可以在 TensorFlow, Jax, 以及其他机器学习框架中稳定使用的数据集集合。TFDS 将下载和准备数据部分包装好，并且构建一个 [tf.data.Dataset](https://www.tensorflow.org/api_docs/python/tf/data/Dataset?hl=zh-cn) 。
可以使用以下两个软件包安装 TFDS：

- pip install tensorflow-datasets：稳定版，数月发行一次。
- pip install tfds-nightly：每天发行，包含最近版本的数据集。

以下是 TFDS 的常用使用方式：
```python
import tensorflow_datasets as tfds

## 查找可用的数据集
tfds.list_builders()

## 加载数据集
ds = tfds.load('mnist', split='train', shuffle_files=True)

## 使用 tfds.builder 加载数据
builder = tfds.builder('mnist')
builder.download_and_prepare()
ds = builder.as_dataset(split='train', shuffle_files=True)

## 对数据集进行基准分析
tfds.benchmark(ds, batch_size=32)

## 展现为数据帧
tfds.as_dataframe(ds.take(4), info)
```
### TFDS CLI
TFDS cli 是一个命令行工具，简化了对 TFDS 的操作。
TFDS cli 会随着 TFDS 安装时一起安装。
获取 cli 命令列表：`tfds --help`
实现一个新数据集：`tfds new my_dataset`
下载并准备数据集：`tfds build <my_dataset>`
### Splits and slicing
Slicing 指令在 tfds.load 中使用，如下：
```python
ds = tfds.load('my_dataset', split='train[:75%]')
```
Split 有以下的功能：

- 划分训练集和测试集
- 切片：
   - 绝对值切片：`'train[123:450]', train[:4000]`
   - 百分比切片：`'train[:75%]', 'train[25%:75%]'`
   - shard：`train[:4shard], train[4shard]`
- Union of splits：`'train+test', 'train[:25%]+test'`
- 全数据集：**'all'**
- 列表分割：**['train', 'test']**

tfds.even_splits 
```python
split0, split1, split2 = tfds.even_splits('train', n=3)

ds = tfds.load('my_dataset', split=split2)
```
Slicing and metadata
```python
builder = tfds.builder('my_dataset')
builder.info.splits['train'].num_examples  # 10_000
builder.info.splits['train[:75%]'].num_examples  # 7_500 (also works with slices)
builder.info.splits.keys()  # ['train', 'test']
```
Cross validation
```python
vals_ds = tfds.load('mnist', split=[
    f'train[{k}%:{k+10}%]' for k in range(0, 100, 10)
])
trains_ds = tfds.load('mnist', split=[
    f'train[:{k}%]+train[{k+10}%:]' for k in range(0, 100, 10)
])
```
### 性能提示
对数据集进行基准分析
```python
ds = tfds.load('mnist', split='train').batch(32).prefetch()
# Display some benchmark statistics
tfds.benchmark(ds, batch_size=32)
# Second iteration is much faster, due to auto-caching
tfds.benchmark(ds, batch_size=32)
```
缓存数据集：TFDS 会自动对小数据集进行缓存，加载进内存中。
### FeatureConnector
tfds.features.FeatureConnector 定义数据集特征结构
```python
tfds.core.DatasetInfo(
    features=tfds.features.FeaturesDict({
        'image': tfds.features.Image(shape=(28, 28, 1), doc='Grayscale image'),
        'label': tfds.features.ClassLabel(
            names=['no', 'yes'],
            doc=tfds.features.Documentation(
                desc='Whether this is a picture of a cat',
                value_range='yes or no'
            ),
        ),
        'metadata': {
            'id': tf.int64,
            'timestamp': tfds.features.Scalar(
                tf.int64,
                doc='Timestamp when this picture was taken as seconds since epoch'),
            'language': tf.string,
        },
    }),
)
```
序列化和反序列化
```python
with tf.io.TFRecordWriter('path/to/file.tfrecord') as writer:
  for ex in all_exs:
    ex_bytes = features.serialize_example(data)
    f.write(ex_bytes)

ds = tf.data.TFRecordDataset('path/to/file.tfrecord')
ds = ds.map(features.deserialize_example)
```
访问元数据
```python
ds, info = tfds.load(..., with_info=True)

info.features['label'].names  # ['cat', 'dog', ...]
info.features['label'].str2int('cat')  # 0
```
自定义 FeatureConnector（[https://www.tensorflow.org/datasets/api_docs/python/tfds/features/FeatureConnector](https://www.tensorflow.org/datasets/api_docs/python/tfds/features/FeatureConnector)）

- 继承并实现 tfds.features.FeatureConnector 中的抽象方法。
- encode_example(data)：定义如何将在生成器 _generate_examples() 中给定的数据编码成兼容 [tf.train.Example](https://www.tensorflow.org/api_docs/python/tf/train/Example?hl=zh-cn) 的数据。可以返回单个值或值的 dict。
- decode_example(data)：定义如何将从 [tf.train.Example](https://www.tensorflow.org/api_docs/python/tf/train/Example?hl=zh-cn) 读取的张量中的数据解码成 [tf.data.Dataset](https://www.tensorflow.org/api_docs/python/tf/data/Dataset?hl=zh-cn) 返回的用户张量。
- get_tensor_info()：指定 [tf.data.Dataset](https://www.tensorflow.org/api_docs/python/tf/data/Dataset?hl=zh-cn) 返回的张量的形状/数据类型。如果从另一个 [tfds.features](https://www.tensorflow.org/datasets/api_docs/python/tfds/features?hl=zh-cn) 继承，则是可选项。
- （可选）get_serialized_info()：如果 get_tensor_info() 返回的信息与实际将数据写入磁盘的方式不同，那么您需要重写 get_serialized_info() 以匹配 [tf.train.Example](https://www.tensorflow.org/api_docs/python/tf/train/Example?hl=zh-cn) 的规范。
- to_json_content/from_json_content：这是允许在没有原始源代码的情况下加载数据集所必需的。
### 创建数据集

- 默认模板

添加新数据集，需要生成一系列 python 文件，下述脚本可以生成所需的 python 文件：
```python
python tensorflow_datasets/scripts/create_new_dataset.py \
  --dataset my_dataset \
  --type image  # text, audio, translation,...
```

- DatasetBuilder

DatasetBuilder 是大多数数据集的父类，有以下几个方法：

   - _info：建立描述数据集的 DatasetInfo 对象。
   - _download_and_prepare：将源数据下载并序列化到磁盘。
   - _as_dataset：从序列化数据中产生一个 tf.data.Dataset。

tfds.core.GeneratorBasedBuilder 是 DatasetBuilder 的一个子类，可以简化定义数据集，其实现的方法如下：

   - _info: 建立描述数据集的 DatasetInfo 对象。
   - _split_generators: 下载源数据并定义数据分割。
   - _generate_examples: 从源数据中产生 (key, example) 元组。
- 指定 DatasetInfo
```cpp
class MyDataset(tfds.core.GeneratorBasedBuilder):

  def _info(self):
    return tfds.core.DatasetInfo(
        builder=self,
        # 这是将在数据集页面上显示的描述。
        description=("This is the dataset for xxx. It contains yyy. The "
                     "images are kept at their original dimensions."),
        # tfds.features.FeatureConnectors
        features=tfds.features.FeaturesDict({
            "image_description": tfds.features.Text(),
            "image": tfds.features.Image(),
            # 在这里，标签可以是5个不同的值。
            "label": tfds.features.ClassLabel(num_classes=5),
        }),
        # 如果特征中有一个通用的（输入，目标）元组，请在此处指定它们。
        # 它们将会在 builder.as_dataset 中的 as_supervised=True 时被使用。
        supervised_keys=("image", "label"),
        # 用于文档的数据集主页
        homepage="https://dataset-homepage.org",
        # 数据集的 Bibtex 引用
        citation=r"""@article{my-awesome-dataset-2020,
                              author = {Smith, John},"}""",
    )
```

- 下载和提取源数据
```cpp
def _split_generators(self, dl_manager):
  # 相当于 dl_manager.extract(dl_manager.download(urls))
  dl_paths = dl_manager.download_and_extract({
      'foo': 'https://example.com/foo.zip',
      'bar': 'https://example.com/bar.zip',
  })
  dl_paths['foo'], dl_paths['bar']
```

- 手动下载和提取

对于不能自动下载的源数据（例如，下载可能需要登陆），可以手动下载源数据并将其放在 `manual_dir` 中，通过 `dl_manager.manual_dir` 访问该文件夹（默认为~/tensorflow_datasets/manual/my_dataset）。

- 指定数据集分割

如果数据集带有预定义的分割（例如，MNSIT 有训练和测试分割），那么就在 DatasetBuilder 中保留那些分割。如果数据集没有预定义的分割，则 DatasetBuilder 应指定一个 tfds.Split.TRAIN 分割。
```cpp
def _split_generators(self, dl_manager):
    # 下载源数据
    extracted_path = dl_manager.download_and_extract(...)

    # 指定分割
    return [
        tfds.core.SplitGenerator(
            name=tfds.Split.TRAIN,
            gen_kwargs={
                "images_dir_path": os.path.join(extracted_path, "train"),
                "labels": os.path.join(extracted_path, "train_labels.csv"),
            },
        ),
        tfds.core.SplitGenerator(
            name=tfds.Split.TEST,
            gen_kwargs={
                "images_dir_path": os.path.join(extracted_path, "test"),
                "labels": os.path.join(extracted_path, "test_labels.csv"),
            },
        ),
    ]
```

- 样本生成器
```cpp
def _generate_examples(self, images_dir_path, labels):
  # 从源文件中读取输入数据
  for image_file in tf.io.gfile.listdir(images_dir_path):
    ...
  with tf.io.gfile.GFile(labels) as f:
    ...

  # 并以特征字典的方式生成样本
  for image_id, description, label in data:
    yield image_id, {
        "image_description": description,
        "image": "%s/%s.jpeg" % (images_dir_path, image_id),
        "label": label,
    }
```

- 文件存储和 tf.io.gfile

为了支持云存储系统，对所有文件系统的访问，使用 tf.io.gfile 或者其他 TensorFlow 文件 APIs（例如，tf.python_io）
