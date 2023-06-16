# Integrate Tensorflow
## 版本记录
| 版本 | 修改时间 | 备注 |
| --- | --- | --- |
| 0.0.1 | 2023/5/27 | 初稿 |
| 0.0.2 | 2023/6/4 | 添加方案设计 |

## 需求分析

- tensorflow serving 支持 dragonfly 分发文件。
## 目标

- 编写 tensorflow 文件系统插件，提供 dragonfly 分发文件的功能。
- 将插件链接进 tensorflow serving 服务中。
- 构建新的镜像文件。
## 技术架构
### tensorflow modular filesystem interface
tensorflow serving 使用 tensorflow 的文件系统实现模型文件加载。 tensorflow modular filesystem 提供了一个可扩展的插件形式，支持定制的文件系统实现。为了使用 dragonfly 的文件分发能力，需要编写一个支持 dragonfly 的 modular filesystem，并链接进 tensorflow serving 中。modular filesystem 规定了需要实现的接口，如下说明所示。
#### 1、插件特定数据
```cpp
// 下面这些数据结构对于 tensorflow 是不透明的 (opaque)
typedef struct TF_RandomAccessFile {
  void* plugin_file;
} TF_RandomAccessFile;

typedef struct TF_WritableFile {
  void* plugin_file;
} TF_WritableFile;

typedef struct TF_ReadOnlyMemoryRegion {
  void* plugin_memory_region;
} TF_ReadOnlyMemoryRegion;

typedef struct TF_Filesystem {
  void* plugin_filesystem;
} TF_Filesystem;

typedef struct TF_TransactionToken {
  void* token;
  TF_Filesystem* owner;
} TF_TransactionToken;

typedef union TF_Filesystem_Option_Value_Union {
  int64_t int_val;
  double real_val;
  struct {
    char* buf;
    int buf_length;
  } buffer_val;
} TF_Filesystem_Option_Value_Union;

typedef struct TF_Filesystem_Option_Value {
  int type_tag;    
  int num_values; 
  TF_Filesystem_Option_Value_Union*
      values;
} TF_Filesystem_Option_Value;

typedef enum TF_Filesystem_Option_Type {
  TF_Filesystem_Option_Type_Int = 0,
  TF_Filesystem_Option_Type_Real,
  TF_Filesystem_Option_Type_Buffer,
  TF_Filesystem_Num_Option_Types,
} TF_Filesystem_Option_Type;

typedef struct TF_Filesystem_Option {
  char* name;                         
  char* description;                  
  int per_file;                       
  TF_Filesystem_Option_Value* value; 
} TF_Filesystem_Option;
```
#### 2、插件函数表
在调用注册函数时，需要将插件中实现的函数全部赋予函数表。总共由 4 种函数表，其中 3 种代表 tensorflow 中的文件对象(`RandomAccessFile`,  `WritableFile`, `ReadOnlyMemoryRegion`)，剩下一种实现 `filesystem`的所有操作。
```cpp
typedef struct TF_RandomAccessFileOps {
  /// 释放关联于 file 的资源
  /// 该方法必须提供
  void (*cleanup)(TF_RandomAccessFile* file);

  /// 从 file 偏移量为 offset 处读取 n 比特数据.
  /// 输出是 buffer
  /// Plugins:
  ///   * 如果可以读取 n 比特数据，设置 status 为 `TF_OK`。
  ///   * 如果可读取数据少于 n，设置 status 为 `TF_OUT_OF_RANGE`。
  ///   * 如果发生错误，必须返回 -1，并且设置 status 错误信息。
  int64_t (*read)(const TF_RandomAccessFile* file, uint64_t offset, size_t n,
                  char* buffer, TF_Status* status);
} TF_RandomAccessFileOps;

typedef struct TF_WritableFileOps {
	void (*cleanup)(TF_WritableFile* file);

	/// 添加 n 比特数据到 file 中.
  	/// Plugins:
  	///   * 如果成功写 n 比特数据，设置 status 为 `TF_OK`。
  	///   * 如果写数据少于 n，设置 status 为 `TF_RESOURCE_EXHAUSTED`。
  	///   * 若发生错误，设置 status 错误信息。
	void (*append)(const TF_WritableFile* file, const char* buffer, size_t n,
                 TF_Status* status);

	/// 返回当前写的位置
	/// Plugins:
  	///   * 成功设置 status 为 `TF_OK`，并返回位置。
  	///   * 如果发生错误，必须返回 -1，并且设置 status 错误信息。
	int64_t (*tell)(const TF_WritableFile* file, TF_Status* status);

	/// 刷新 file，并同步内容到文件系统
    /// DEFAULT IMPLEMENTATION: No op.
	void (*flush)(const TF_WritableFile* file, TF_Status* status);

	/// 同步文件系统内容
	/// DEFAULT IMPLEMENTATION: No op.
    void (*sync)(const TF_WritableFile* file, TF_Status* status);

	/// 关闭文件。
	void (*close)(const TF_WritableFile* file, TF_Status* status);
} TF_WritableFileOps;

typedef struct TF_ReadOnlyMemoryRegionOps {
	void (*cleanup)(TF_ReadOnlyMemoryRegion* region);

	/// 返回一个指向内存区域的指针
	const void* (*data)(const TF_ReadOnlyMemoryRegion* region);

	/// 返回内存中数据的长度
	uint64_t (*length)(const TF_ReadOnlyMemoryRegion* region);
} TF_ReadOnlyMemoryRegionOps;

typedef struct TF_FilesystemOps {
	/// 获取所有 filesystem 必须的资源
	/// 必须实现
	void (*init)(TF_Filesystem* filesystem, TF_Status* status);

	void (*cleanup)(TF_Filesystem* filesystem);

	/// 基于给定的 path 创建一个新的 RandomAccessFile 对象。
	/// 如果成功，设置 status 为 `TF_OK`。
	/// 如果路径不存在，设置 status 为 `TF_NOT_FOUND`。
	/// 如果路径是一个目录或无效路径，设置 status 为 `TF_FAILED_PRECONDITION`。
	/// 如果失败，设置 status 为错误信息。
	void (*new_random_access_file)(const TF_Filesystem* filesystem,
                                 const char* path, TF_RandomAccessFile* file,
                                 TF_Status* status);

	/// 基于给定的 path 创建一个新的 WritableFile 对象。
	/// 如果成功，设置 status 为 `TF_OK`。
	/// 如果路径不存在，设置 status 为 `TF_NOT_FOUND`。
	/// 如果路径是一个目录或无效路径，设置 status 为 `TF_FAILED_PRECONDITION`。
	/// 如果失败，设置 status 为错误信息。
	void (*new_writable_file)(const TF_Filesystem* filesystem, const char* path,
                            TF_WritableFile* file, TF_Status* status);

	/// 基于给定的 path 创建一个新的 appendable_file 对象。
	/// 如果成功，设置 status 为 `TF_OK`。
	/// 如果路径不存在，设置 status 为 `TF_NOT_FOUND`。
	/// 如果路径是一个目录或无效路径，设置 status 为 `TF_FAILED_PRECONDITION`。
	/// 如果失败，设置 status 为错误信息。
	void (*new_appendable_file)(const TF_Filesystem* filesystem, const char* path,
                              TF_WritableFile* file, TF_Status* status);
	
	/// 基于给定的 path 创建一个 read-only region of memory。
	/// 如果成功，设置 status 为 `TF_OK`。
	/// 如果路径不存在，设置 status 为 `TF_NOT_FOUND`。
	/// 如果路径是一个目录或无效路径，设置 status 为 `TF_FAILED_PRECONDITION`。
	/// 如果路径指向一个空目录，设置 status 为 `TF_INVALID_ARGUMENT`。
	/// 如果失败，设置 status 为错误信息。
    void (*new_read_only_memory_region_from_file)(const TF_Filesystem* filesystem,
                                                const char* path,
                                                TF_ReadOnlyMemoryRegion* region,
                                              TF_Status* status);

	/// 创建一个目录
	/// 如果成功，设置 status 为 `TF_OK`。
	/// 如果路径不存在，设置 status 为 `TF_NOT_FOUND`。
	/// 如果路径是一个目录或无效路径，设置 status 为 `TF_FAILED_PRECONDITION`。
	/// 如果路径已存在，设置 status 为 `TF_ALREADY_EXISTS`。
	/// 如果失败，设置 status 为错误信息。
	void (*create_dir)(const TF_Filesystem* filesystem, const char* path,
                     TF_Status* status);

	/// 创建一个目录和所有需要的上级目录
	void (*recursively_create_dir)(const TF_Filesystem* filesystem,
                                 const char* path, TF_Status* status);
	
	/// 删除文件
	void (*delete_file)(const TF_Filesystem* filesystem, const char* path,
                      TF_Status* status);

	/// 删除目录
	void (*delete_dir)(const TF_Filesystem* filesystem, const char* path,
                     TF_Status* status);

	/// 删除目录和其下所有的内容
	void (*delete_recursively)(const TF_Filesystem* filesystem, const char* path,
                             uint64_t* undeleted_files,
                             uint64_t* undeleted_dirs, TF_Status* status);

	/// 重命名文件
	void (*rename_file)(const TF_Filesystem* filesystem, const char* src,
                      const char* dst, TF_Status* status);

	/// 复制文件
	void (*copy_file)(const TF_Filesystem* filesystem, const char* src,
                    const char* dst, TF_Status* status);

	/// 检查目录是否存在
	void (*path_exists)(const TF_Filesystem* filesystem, const char* path,
                      TF_Status* status);

	/// 检查是否所有目录存在
	bool (*paths_exist)(const TF_Filesystem* filesystem, char** paths,
                      int num_files, TF_Status** statuses);

	/// 获取文件统计
	void (*stat)(const TF_Filesystem* filesystem, const char* path,
               TF_FileStatistics* stats, TF_Status* status);

	/// 检查给定路径是否是目录
	bool (*is_directory)(const TF_Filesystem* filesystem, const char* path,
                       TF_Status* status);

	/// 返回文件大小
	int64_t (*get_file_size)(const TF_Filesystem* filesystem, const char* path,
                           TF_Status* status);

	/// 解析路径
	char* (*translate_name)(const TF_Filesystem* filesystem, const char* uri);

	/// 获取子路径
	int (*get_children)(const TF_Filesystem* filesystem, const char* path,
                      char*** entries, TF_Status* status);

    /// 匹配所有满足正则表达式的路径
	int (*get_matching_paths)(const TF_Filesystem* filesystem, const char* glob,
                            char*** entries, TF_Status* status);
	
	/// 刷新在内存中的文件系统缓存
	void (*flush_caches)(const TF_Filesystem* filesystem);

	int (*start_transaction)(const TF_Filesystem* filesystem,
                           TF_TransactionToken** token, TF_Status* status);

	int (*end_transaction)(const TF_Filesystem* filesystem,
                         TF_TransactionToken* token, TF_Status* status);
	
	int (*add_to_transaction)(const TF_Filesystem* filesystem, const char* path,
                            TF_TransactionToken* token, TF_Status* status);

	int (*get_transaction_for_path)(const TF_Filesystem* filesystem,
                                  const char* path, TF_TransactionToken** token,
                                  TF_Status* status);

	int (*get_or_start_transaction_for_path)(const TF_Filesystem* filesystem,
                                           const char* path,
                                           TF_TransactionToken** token,
                                           TF_Status* status);

	char* (*decode_transaction_token)(const TF_Filesystem* filesystem,
                                    const TF_TransactionToken* token);

	/// 返回配置选项
	void (*get_filesystem_configuration)(const TF_Filesystem* filesystem,
                                       TF_Filesystem_Option** options,
                                       int* num_options, TF_Status* status);

	/// 设置配置选项
	void (*set_filesystem_configuration)(const TF_Filesystem* filesystem,
                                       const TF_Filesystem_Option* options,
                                       int num_options, TF_Status* status);

	/// 根据 key 获取配置
	void (*get_filesystem_configuration_option)(const TF_Filesystem* filesystem,
                                              const char* key,
                                              TF_Filesystem_Option** option,
                                              TF_Status* status);

	void (*set_filesystem_configuration_option)(
      const TF_Filesystem* filesystem, const TF_Filesystem_Option* option,
      TF_Status* status);

	void (*get_filesystem_configuration_keys)(const TF_Filesystem* filesystem,
                                            char** keys, int* num_keys,
                                            TF_Status* status);
} TF_FilesystemOps;
```
#### 3、 ABI and API 兼容
```cpp
constexpr int TF_RANDOM_ACCESS_FILE_OPS_API = 0;
constexpr int TF_RANDOM_ACCESS_FILE_OPS_ABI = 0;
constexpr size_t TF_RANDOM_ACCESS_FILE_OPS_SIZE =
    sizeof(TF_RandomAccessFileOps);

constexpr int TF_WRITABLE_FILE_OPS_API = 0;
constexpr int TF_WRITABLE_FILE_OPS_ABI = 0;
constexpr size_t TF_WRITABLE_FILE_OPS_SIZE = sizeof(TF_WritableFileOps);

constexpr int TF_READ_ONLY_MEMORY_REGION_OPS_API = 0;
constexpr int TF_READ_ONLY_MEMORY_REGION_OPS_ABI = 0;
constexpr size_t TF_READ_ONLY_MEMORY_REGION_OPS_SIZE =
    sizeof(TF_ReadOnlyMemoryRegionOps);

constexpr int TF_FILESYSTEM_OPS_API = 0;
constexpr int TF_FILESYSTEM_OPS_ABI = 0;
constexpr size_t TF_FILESYSTEM_OPS_SIZE = sizeof(TF_FilesystemOps);
```
#### 4、注册
```cpp
typedef struct TF_FilesystemPluginOps {
  char* scheme;
  int filesystem_ops_abi;
  int filesystem_ops_api;
  size_t filesystem_ops_size;
  TF_FilesystemOps* filesystem_ops;
  int random_access_file_ops_abi;
  int random_access_file_ops_api;
  size_t random_access_file_ops_size;
  TF_RandomAccessFileOps* random_access_file_ops;
  int writable_file_ops_abi;
  int writable_file_ops_api;
  size_t writable_file_ops_size;
  TF_WritableFileOps* writable_file_ops;
  int read_only_memory_region_ops_abi;
  int read_only_memory_region_ops_api;
  size_t read_only_memory_region_ops_size;
  TF_ReadOnlyMemoryRegionOps* read_only_memory_region_ops;
} TF_FilesystemPluginOps;

typedef struct TF_FilesystemPluginInfo {
  size_t num_schemes;
  TF_FilesystemPluginOps* ops;
  void* (*plugin_memory_allocate)(size_t size);
  void (*plugin_memory_free)(void* ptr);
} TF_FilesystemPluginInfo;

static inline void TF_SetFilesystemVersionMetadata(
    TF_FilesystemPluginOps* ops) {
  ops->filesystem_ops_abi = TF_FILESYSTEM_OPS_ABI;
  ops->filesystem_ops_api = TF_FILESYSTEM_OPS_API;
  ops->filesystem_ops_size = TF_FILESYSTEM_OPS_SIZE;
  ops->random_access_file_ops_abi = TF_RANDOM_ACCESS_FILE_OPS_ABI;
  ops->random_access_file_ops_api = TF_RANDOM_ACCESS_FILE_OPS_API;
  ops->random_access_file_ops_size = TF_RANDOM_ACCESS_FILE_OPS_SIZE;
  ops->writable_file_ops_abi = TF_WRITABLE_FILE_OPS_ABI;
  ops->writable_file_ops_api = TF_WRITABLE_FILE_OPS_API;
  ops->writable_file_ops_size = TF_WRITABLE_FILE_OPS_SIZE;
  ops->read_only_memory_region_ops_abi = TF_READ_ONLY_MEMORY_REGION_OPS_ABI;
  ops->read_only_memory_region_ops_api = TF_READ_ONLY_MEMORY_REGION_OPS_API;
  ops->read_only_memory_region_ops_size = TF_READ_ONLY_MEMORY_REGION_OPS_SIZE;
}

TF_CAPI_EXPORT extern void TF_InitPlugin(TF_FilesystemPluginInfo* plugin_info);
```
### dragonfly 代理 tensorflow 文件系统
![图片1.png](https://cdn.nlark.com/yuque/0/2023/png/35903190/1685794723179-0cb2d9ff-826b-49d1-be3a-1861c2a63bd9.png#averageHue=%23a7cf8d&clientId=ub909a7a1-908b-4&from=ui&id=u23cf2941&originHeight=744&originWidth=1262&originalType=binary&ratio=1.5&rotation=0&showTitle=false&size=23134&status=done&style=none&taskId=ubfb560d7-67db-4594-a11a-12b0fd9c968&title=)

- tensorflow model serving 初始化时加载编写的 tensorflow 文件系统插件，并注册进 tensorflow rt，使用 dragonfly 作为注册的标识。
- 若使用 dragonfly 分发文件，在 tensorflow model serving 中的 model_base_path 指定路径时，使用 dragonfly 前缀，例如 dragonfly://path/to 。
- 指定的路径应是 dragonfly 的配置文件或含有配置文件。
- tensorflow model serving 请求含有 dragonfly:// 前缀路径的文件时，tensorflow runtime 从注册的所有文件系统实现中选出 dragonfly 插件实现。在初始化时，从文件中读取 dragonfly 配置文件，并将请求转交给 dragonfly。
### tensorflow serving 镜像
tensorflow serving 是一个高度模块化的应用，支持用户定制一个 model serving 应用，但是其并没有提供一个即插即用的插件加载机制，因此需要编写额外的代码并重构 tensorflow serving 应用。目前的功能如下：

- 通过环境变量的方式传入一个动态库地址，环境变量命名示例：DRAGONFLY_PLUGIN_PATH 。
- 编写一个静态方法和静态实例，以在 tensorflow serving 启动时初始化。方法需要读取环境变量，并加载对应的动态库文件。
### tensorflow 文件系统插件
该插件主要在原有的文件系统上使用 dragonfly 作为代理，由 dragonfly 执行实际的对文件的操作。主要实现以下的功能：

- 如果代理的地址是一个本地文件系统，dragonfly 不做任何处理。
- 如果代理的地址是对象存储系统，文件分发工作交给 dragonfly ，然后返回一个本地的临时路径地址。
- dragonfly 的配置文件在 tensorflow serving 的配置中指定。 model_base_path 选项指定一个 dragonfly 的配置文件，原有的地址写在该配置文件中。插件负责读取该配置文件并读取出真正的地址，交给 dragonfly 处理。
## 方案设计
### 目录结构
```
.
└── filesystems
    ├── dragonfly
    │   ├── dragonfly_filesystem.cc
    │   └── dragonfly_filesystem.h
    ├── dragonfly_plugins.cc
    └── dragonfly_plugins.h
```
### 配置文件
为了将所有的更改保持在插件中，使用环境变量的方式将 dragonfly 所必须的所有配置信息传入到插件中。环境变量的命名方式为 DRAGONFLY_CONFIG_PATH ，其值是用户指定的一个配置文件本地路径，支持相对路径和绝对路径。为保持一致性，配置文件格式与 dragonfly 保持一致。样例如下：
```bash
export DRAGONFLY_CONFIG_PATH=[example.json | /home/example.json]
// or
docker -e DRAGONFLY_CONFIG_PATH=[example.json | /home/example.json]

// example.json
{
	...
}
```
### 插件设计
参考 [tensorflow/io](https://github.com/tensorflow/io/tree/master/tensorflow_io/core/filesystems) 项目中插件的实现，主要将 dragonfly 作为访问实际文件存储的客户端，相当于一个文件访问的中间代理。基于 tensorflow modular filesystem 定义的接口，主要实现 dragonfly_filesystem.h 和 dragonfly_filesystem.cc 文件。其中 dragonfly_filesystem.h 声明需要实现的接口方法，与 tensorflow modular filesystem 中定义的方法一致。dragonfly_filesystem.cc 主要实现如下：
```cpp
/*	Copyright message
	...
==============================================================================*/

# include "tensorflow_io/core/filesystems/dragonfly/dragonfly_filesystem.h"

namespace tensorflow {
namespace io {
namespace dragonfly {
// Implementation of a filesystem proxy using dragonfly.
// This plugin will support `dragonfly//` URI schemes.

// handle error type  
static inline void TF_SetStatusFromDragonflyError (DragonflyError ,TF_Status* status) {
    // TODO
}

// dragonfly configure options
typedef struct Options {
	// TODO: add required configuration fields

	// if options has be initialized 
	bool configured = false;
} Options;

/// init the options
/// if success, return TF_OK
/// else, return TF_NOT_FOUND with error message
void create_options(Options* options, TF_Status status) {
    /// 1. get dragonfly config file from env
    /// 2. parse the file and fill the options
}

// SECTION 1. Implementation for `TF_RandomAccessFile`
// ----------------------------------------------------------------------------
namespace tf_random_access_file {
	// abstraction of dragonfly
	// some required data to use dragonfly
	typedef struct DragonflyClient {
    	std::shared_ptr<Options> options;
    	// TODO others
    } DragonflyClient;

	void Cleanup(TF_RandomAccessFile* file) {
  		auto df_client = static_cast<DragonflyClient*>(file->plugin_file);
  		delete df_client;
	}

	int64_t ReadFromDragonfly(DragonflyClient* df_client, uint64_t offset, size_t n,
						char* buffer, TF_Status* status) {
        // TODO
        // 1、get required info of dragonfly.
        // 2、send a request to dragonfly.
        // 3、if dragonfly return the buffer directly, return TF_OK.
        // 	  or download the file first, than read file by local filesystem.
    }

	// read `n` bytes from `file`
	int64_t Read(const TF_RandomAccessFile* file, uint64_t offset, size_t n,
             char* buffer, TF_Status* status) {
        auto df_client = static_cast<DragonflyClient*>(file->plugin_file);
        // TODO: some log
        ReadFromDragonfly(df_client, offset, n, buffer, status);
    }
} // namespace tf_random_access_file

namespace tf_writable_file {
	typedef struct DragonflyClient {
    	std::shared_ptr<Options> options;
    	// TODO others
    } DragonflyClient;

	void Cleanup(TF_RandomAccessFile* file) {
  		auto df_client = static_cast<DragonflyClient*>(file->plugin_file);
  		delete df_client;
	}

	// append `n` bytes to file
	void Append(const TF_WritableFile* file, const char* buffer, size_t n,
            TF_Status* status) {
        // TODO
        // Maybe dragonfly can't support this. If yes, return TF_Unknow directly.
        // If not, write n byte to tmp file, and call `Sync` to flush. 
    }

	// return current write location
	int64_t Tell(const TF_WritableFile* file, TF_Status* status) {
        // TODO
        // similar to `Append`
    }

	void Sync(const TF_WritableFile* file, TF_Status* status) {
        // TODO
        // if call `Append` method, this should be called. Send tmp file to dragonfly.
        // 1、Check if synchronization is required.
        // 2、If needed, send a upload requset to dragonfly.
        // 3、If failed, retry some times. Return TF_UNKNOWN with error message.
    }

	void Flush(const TF_WritableFile* file, TF_Status* status) {
  		Sync(file, status);
	} 

	void Close(const TF_WritableFile* file, TF_Status* status) {
        // TODO
        // free DragonflyClient
        // There should call `Sync`.
    }
} // namespace tf_writable_file

// SECTION 3. Implementation for `TF_ReadOnlyMemoryRegion`
// ----------------------------------------------------------------------------
namespace tf_read_only_memory_region {
	void Cleanup(TF_ReadOnlyMemoryRegion* region) {
        
    }

	const void* Data(const TF_ReadOnlyMemoryRegion* region) {
        
    }

	uint64_t Length(const TF_ReadOnlyMemoryRegion* region) {
        
    }
} // namespace tf_read_only_memory_region

// SECTION 4. Implementation for `TF_Filesystem`, the actual filesystem
// ----------------------------------------------------------------------------
namespace tf_dragonfly_filesystem {
	typedef struct DragonflyClient {
    	std::shared_ptr<Options> options;
    	// TODO others
    } DragonflyClient;

	void Init(TF_Filesystem* filesystem, TF_Status* status) {
  		DragonflyClient* client = new DragonflyClient();
        create_options(client.options.get(), status);
        if (TF_GetCode(status) == TF_OK) {
            filesystem->>plugin_filesystem = client;
        }
	}

	void Cleanup(TF_RandomAccessFile* file) {
  		auto df_client = static_cast<DragonflyClient*>(file->plugin_file);
  		delete df_client;
	}

	// return a instance of RandomAccessFile
	void NewRandomAccessFile(const TF_Filesystem* filesystem, const char* path,
                         TF_RandomAccessFile* file, TF_Status* status) {
        // TODO
        // 1、parse path based on the configuration
        // 2、file->plugin_file = new tf_random_access_file::DragonflyClient()
    }

	// return a instance of WritableFile
	void NewWritableFile(const TF_Filesystem* filesystem, const char* path,
                     TF_WritableFile* file, TF_Status* status) {
        // TODO
        // 1、parse path based on the configuration
        // 2、file->plugin_file = new tf_writable_file::DragonflyClient()
    }

	// return a instance of WritableFile base on exist file
	void NewAppendableFile(const TF_Filesystem* filesystem, const char* path,
                       TF_WritableFile* file, TF_Status* status) {
        // TODO
        // 1、create a `WritableFile` with tmp path.
        // 2、create a `RandomAccessFile` with `path`.
        // 3、copy content of `RandomAccessFile` to `WritableFile`.
        // 4、return `WritableFile`.
    }

	// Obtains statistics for the given `path`.
	void Stat(const TF_Filesystem* filesystem, const char* path,
          TF_FileStatistics* stats, TF_Status* status) {
        // `TF_FileStatistics` has three field:
        // 		length: The length of the file in bytes.
        //		mtime_nsec: The last modified time in nanoseconds.
        //		is_directory: Whether the name refers to a directory.
        // TODO: 
        // fill the `stats` according to the give `path`.
    }

	void PathExists(const TF_Filesystem* filesystem, const char* path,
                TF_Status* status) {
  		TF_FileStatistics stats;
  		Stat(filesystem, path, &stats, status);
	}
	
	int64_t GetFileSize(const TF_Filesystem* filesystem, const char* path,
                    TF_Status* status) {
  		TF_FileStatistics stats;
  		Stat(filesystem, path, &stats, status);
  		return stats.length;
	}

	void NewReadOnlyMemoryRegionFromFile(const TF_Filesystem* filesystem,
                                     const char* path,
                                     TF_ReadOnlyMemoryRegion* region,
                                     TF_Status* status) {
        // TODO
        // 1、create a `RandomAccessFile` with `path`.
        // 2、read file from `RandomAccessFile` add write to memory.
    }

	void CopyFile(const TF_Filesystem* filesystem, const char* src, const char* dst,
              TF_Status* status) {
        // TODO
        // if Dragonfly supports
    }

	void DeleteFile(const TF_Filesystem* filesystem, const char* path,
                TF_Status* status) {
        // TODO
        // if Dragonfly supports
    }

	void CreateDir(const TF_Filesystem* filesystem, const char* path,
               TF_Status* status) {
        // TODO
        // 1、check path.
        // 2、create `WritableFile`.
    }

	void RecursivelyCreateDir(const TF_Filesystem* filesystem, const char* path,
                          TF_Status* status) {
  		CreateDir(filesystem, path, status);
	}

	void DeleteDir(const TF_Filesystem* filesystem, const char* path,
               TF_Status* status) {
        // TODO
        // if Dragonfly supports
    }

	void RenameFile(const TF_Filesystem* filesystem, const char* src,
                const char* dst, TF_Status* status) {
        // TODO
        // if Dragonfly supports
    }

    int GetChildren(const TF_Filesystem* filesystem, const char* path,
                char*** entries, TF_Status* status) {
        // TODO
        // if Dragonfly supports
    }

	static char* TranslateName(const TF_Filesystem* filesystem, const char* uri) {
  		return strdup(uri);
	}
	
} // namespace tf_dragonfly_filesystem
void ProvideFilesystemSupportFor(TF_FilesystemPluginOps* ops, const char* uri) {
  TF_SetFilesystemVersionMetadata(ops);
  ops->scheme = strdup(uri);

  ops->random_access_file_ops = static_cast<TF_RandomAccessFileOps*>(
      plugin_memory_allocate(TF_RANDOM_ACCESS_FILE_OPS_SIZE));
  ops->random_access_file_ops->cleanup = tf_random_access_file::Cleanup;
  ops->random_access_file_ops->read = tf_random_access_file::Read;

  ops->writable_file_ops = static_cast<TF_WritableFileOps*>(
      plugin_memory_allocate(TF_WRITABLE_FILE_OPS_SIZE));
  ops->writable_file_ops->cleanup = tf_writable_file::Cleanup;
  ops->writable_file_ops->append = tf_writable_file::Append;
  ops->writable_file_ops->tell = tf_writable_file::Tell;
  ops->writable_file_ops->flush = tf_writable_file::Flush;
  ops->writable_file_ops->sync = tf_writable_file::Sync;
  ops->writable_file_ops->close = tf_writable_file::Close;

  ops->read_only_memory_region_ops = static_cast<TF_ReadOnlyMemoryRegionOps*>(
      plugin_memory_allocate(TF_READ_ONLY_MEMORY_REGION_OPS_SIZE));
  ops->read_only_memory_region_ops->cleanup =
      tf_read_only_memory_region::Cleanup;
  ops->read_only_memory_region_ops->data = tf_read_only_memory_region::Data;
  ops->read_only_memory_region_ops->length = tf_read_only_memory_region::Length;

  ops->filesystem_ops = static_cast<TF_FilesystemOps*>(
      plugin_memory_allocate(TF_FILESYSTEM_OPS_SIZE));
  ops->filesystem_ops->init = tf_dragonfly_filesystem::Init;
  ops->filesystem_ops->cleanup = tf_dragonfly_filesystem::Cleanup;
  ops->filesystem_ops->new_random_access_file =
      tf_dragonfly_filesystem::NewRandomAccessFile;
  ops->filesystem_ops->new_writable_file = tf_dragonfly_filesystem::NewWritableFile;
  ops->filesystem_ops->new_appendable_file =
      tf_dragonfly_filesystem::NewAppendableFile;
  ops->filesystem_ops->new_read_only_memory_region_from_file =
      tf_dragonfly_filesystem::NewReadOnlyMemoryRegionFromFile;
  ops->filesystem_ops->create_dir = tf_s3_filesystem::CreateDir;
  ops->filesystem_ops->recursively_create_dir =
      tf_dragonfly_filesystem::RecursivelyCreateDir;
  ops->filesystem_ops->delete_file = tf_dragonfly_filesystem::DeleteFile;
  ops->filesystem_ops->delete_dir = tf_dragonfly_filesystem::DeleteDir;
  ops->filesystem_ops->copy_file = tf_dragonfly_filesystem::CopyFile;
  ops->filesystem_ops->rename_file = tf_dragonfly_filesystem::RenameFile;
  ops->filesystem_ops->path_exists = tf_dragonfly_filesystem::PathExists;
  ops->filesystem_ops->get_file_size = tf_dragonfly_filesystem::GetFileSize;
  ops->filesystem_ops->stat = tf_dragonfly_filesystem::Stat;
  ops->filesystem_ops->get_children = tf_dragonfly_filesystem::GetChildren;
  ops->filesystem_ops->translate_name = tf_dragonfly_filesystem::TranslateName;
}
} // namespace dragonfly
} // namespace io
} // namespace tensorflow

```
