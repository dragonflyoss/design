# Integrate Triton Server
## 版本记录
| 版本 | 修改时间 | 备注 |
| --- | --- | --- |
| 0.0.1 | 2023.05.28 | 第一版，初稿 |
| 0.0.2 | 2023.06.06 | 添加方案详细设计 |

## 需求分析

- Triton Server 支持使用 Dragonfly 分发文件。
## 目标

- 编写 Triton Repoagent 动态库，实现 Triton Server 调用 Dragonfly 加速模型拉取。
- 编译动态库，在模型配置文件中使用。
## 技术架构
### 设计图
![image.png](https://cdn.nlark.com/yuque/0/2023/png/22094377/1685413676317-df118b1d-81bd-442a-949d-8d154e8116d3.png#averageHue=%23f7f9ef&clientId=u5e72f31b-0f82-4&from=paste&height=576&id=ufab0ffaa&originHeight=1152&originWidth=1540&originalType=binary&ratio=2&rotation=0&showTitle=false&size=265295&status=done&style=none&taskId=u070a3e07-1a51-4817-995c-074187e0331&title=&width=770)
灰色虚线为 Triton 自带的从 S3 存储拉取模型文件的步骤，紫色实线为本方案设计的使用 Dragonfly 从 S3 存储拉取模型文件的步骤。
### 初步思路

1. 启动 Triton 时，用户仍然使用 tritonserver --model-repository=s3://path/to/repo 来指定模型仓库的实际地址。
2. Triton 自身解析仓库地址使用的协议（如 s3://），新建 S3 客户端（该客户端仅负责读取位于 S3 上的模型配置文件），使用 ReadTextProto() 函数读取模型配置文件信息，解析出其中指定的 model_repository_agents 字段，加载对应的 repoagent 动态库（/opt/tritonserver/repoagents/d7y/libtritonrepoagent_d7y.so）。
3. Triton 调用 repoagent 实现的 TRITONREPOAGENT_ModelAction() 函数，函数将传入初始的模型地址（s3://path/to/model）以及 Dragonfly 的配置文件，并新建 Dragonfly 客户端，在完成模型拉取后传出新的本地模型仓库的地址。具体如下：
```json
model_repository_agents
{
  agents [
    {
      name: "d7y",
      parameters [
        {
          key: "config_path",
          value: "d7y_config.json" ## 相对地址，与 d7y_agent 在同一级目录
        }
      ]
    }
  ]
}
```
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

  // 通过 ModelRepositoryLocation 传入模型初始地址 location s3://
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

  // 通过 ModelParameterCount 传入 d7y 配置文件信息 [# TODO：从本地环境变量读]
  // TODO：用环境变量的方式传入是否更好？Triton_Dragonfly_Config_Path
  // 全部使用 Dragonfly
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
### RepoAgent API
定义在 tritonrepoagent.h 文件里：
```cpp
// 文件系统格式
typedef enum TRITONREPOAGENT_artifacttype_enum {
  TRITONREPOAGENT_ARTIFACT_FILESYSTEM,
  TRITONREPOAGENT_ARTIFACT_REMOTE_FILESYSTEM
} TRITONREPOAGENT_ArtifactType;

// repoagent 执行动作的时机
typedef enum TRITONREPOAGENT_actiontype_enum {
  TRITONREPOAGENT_ACTION_LOAD,
  TRITONREPOAGENT_ACTION_LOAD_COMPLETE,
  TRITONREPOAGENT_ACTION_LOAD_FAIL,
  TRITONREPOAGENT_ACTION_UNLOAD,
  TRITONREPOAGENT_ACTION_UNLOAD_COMPLETE
} TRITONREPOAGENT_ActionType;

// 获取模型的地址，此地址归属于 Triton 而非 agent，因此不能被更改或者删除
/// \param agent The agent.
/// \param model The model.
/// \param artifact_type Returns the artifact type for the location.
/// \param path Returns the location.
/// \return a TRITONSERVER_Error indicating success or failure.
TRITONREPOAGENT_DECLSPEC TRITONSERVER_Error*
TRITONREPOAGENT_ModelRepositoryLocation(
    TRITONREPOAGENT_Agent* agent, TRITONREPOAGENT_AgentModel* model,
    TRITONREPOAGENT_ArtifactType* artifact_type, const char** location);

// 获得一个新的临时模型仓库的地址用于 agent 对模型作出的更改，该地址归属于 agent，agent 在其生命周期终止时负责调用 TRITONREPOAGENT_ModelRepositoryLocationDelete 删除这个地址
/// \param agent The agent.
/// \param model The model.
/// \param artifact_type The artifact type for the location.
/// \param path Returns the location.
/// \return a TRITONSERVER_Error indicating success or failure.
TRITONREPOAGENT_DECLSPEC TRITONSERVER_Error*
TRITONREPOAGENT_ModelRepositoryLocationAcquire(
    TRITONREPOAGENT_Agent* agent, TRITONREPOAGENT_AgentModel* model,
    const TRITONREPOAGENT_ArtifactType artifact_type, const char** location);

// 释放之前获得的一个模型仓库的地址及其内部的内容
/// \param agent The agent.
/// \param model The model.
/// \param path The location to release.
/// \return a TRITONSERVER_Error indicating success or failure.
TRITONREPOAGENT_DECLSPEC TRITONSERVER_Error*
TRITONREPOAGENT_ModelRepositoryLocationRelease(
    TRITONREPOAGENT_Agent* agent, TRITONREPOAGENT_AgentModel* model,
    const char* location);

// 更新模型仓库的地址，仅在 TRITONREPOAGENT_ACTION_LOAD 时使用，此地址被移交 Triton，agent 无法修改此地址及其内容
/// \param agent The agent.
/// \param model The model.
/// \param artifact_type The artifact type for the location.
/// \param path Returns the location.
/// \return a TRITONSERVER_Error indicating success or failure.
TRITONREPOAGENT_DECLSPEC TRITONSERVER_Error*
TRITONREPOAGENT_ModelRepositoryUpdate(
    TRITONREPOAGENT_Agent* agent, TRITONREPOAGENT_AgentModel* model,
    const TRITONREPOAGENT_ArtifactType artifact_type, const char* location);

// 获取agent参数个数
/// \param agent The agent.
/// \param model The model.
/// \param count Returns the number of input tensors.
/// \return a TRITONSERVER_Error indicating success or failure.
TRITONREPOAGENT_DECLSPEC TRITONSERVER_Error*
TRITONREPOAGENT_ModelParameterCount(
    TRITONREPOAGENT_Agent* agent, TRITONREPOAGENT_AgentModel* model,
    uint32_t* count);

// 获取一对agent参数的key和value
/// \param agent The agent.
/// \param model The model.
/// \param index The index of the parameter. Must be 0 <= index <
/// count, where count is the value returned by
/// TRITONREPOAGENT_ModelParameterCount.
/// \param parameter_name Returns the name of the parameter.
/// \param parameter_value Returns the value of the parameter.
/// \return a TRITONSERVER_Error indicating success or failure.
TRITONREPOAGENT_DECLSPEC TRITONSERVER_Error* TRITONREPOAGENT_ModelParameter(
    TRITONREPOAGENT_Agent* agent, TRITONREPOAGENT_AgentModel* model,
    const uint32_t index, const char** parameter_name,
    const char** parameter_value);

// agent对模型仓库进行的操作，必须实现
TRITONREPOAGENT_ISPEC TRITONSERVER_Error* TRITONREPOAGENT_ModelAction(
    TRITONREPOAGENT_Agent* agent, TRITONREPOAGENT_AgentModel* model,
    const TRITONREPOAGENT_ActionType action_type);

```
### 函数调用逻辑图
#### 概览
![](https://cdn.nlark.com/yuque/0/2023/png/22094377/1685339051072-1652f843-14b7-4372-b8d2-aa443dd94e86.png#averageHue=%23f9f9f9&from=url&id=WjVur&originHeight=919&originWidth=1764&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
#### repoagent 部分
![](https://cdn.nlark.com/yuque/0/2023/png/22094377/1684475642501-0b275c83-f0c4-4df8-9a30-939db7eceb4d.png#averageHue=%23f8f8f8&from=url&id=UO0C0&originHeight=731&originWidth=2519&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
#### model_action_fn 部分
![image.png](https://cdn.nlark.com/yuque/0/2023/png/22094377/1685349887003-02d78d69-78c6-488a-bbe3-f3a683b4fe5e.png#averageHue=%23bdcde9&clientId=u61cba17c-27a8-4&from=paste&height=495&id=uaa941341&originHeight=495&originWidth=1502&originalType=binary&ratio=1&rotation=0&showTitle=false&size=41319&status=done&style=none&taskId=u0f3fcac2-58cf-42af-a6db-f4e4eabc464&title=&width=1502)

在使用 repoagent 将模型拉取到本地后，Triton 本身的 LocalizePath() 函数是否会再次拉取模型或者产生一个新的模型副本呢？答案是否定的，当 LocalizePath 发现传入的 path 为本地文件系统路径时，会直接使用这个路径。如下所示：
```cpp
Status
LocalFileSystem::LocalizePath(
    const std::string& path, std::shared_ptr<LocalizedPath>* localized)
{
  // For local file system we don't actually need to download the
  // directory or file. We use it in place.
  localized->reset(new LocalizedPath(path));
  return Status::Success;
}
```
## 方案设计
### 目录结构
```bash
dragonfly-repository-agent
├── build
│   ├── libtritonrepoagent_dragonfly.so
│   └── Makefile
├── cmake
│   └── TritonDemoRepoAgentConfig.cmake.in
├── CMakeLists.txt
├── LICENSE
├── README.md
└── src
    ├── dragonfly.cc
    └── libtritonrepoagent_dragonfly.ldscript

```
```bash
repoagents
├── checksum
│   └── libtritonrepoagent_checksum.so
└── dragonfly
    └── libtritonrepoagent_dragonfly.so
```
### 配置文件
Triton 自身要求的 S3 等云存储的配置文件命名为 cloud_credential.json，通过环境变量的方式导入，环境变量命名为 TRITON_CLOUD_CREDENTIAL_PATH。格式及导入命令如下：
```bash
export TRITON_CLOUD_CREDENTIAL_PATH="cloud_credential.json"

"cloud_credential.json":
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
Dragonfly 的配置文件命名为 dragonfly_config.json，导入格式与 S3 保持一致，通过环境变量的方式进行导入，环境变量命名为 TRITON_DRAGONFLY_CONFIG_PATH。格式及导入命令如下：
```bash
export TRITON_DRAGONFLY_CONFIG_PATH="dragonfly_config.json"

"dragonfly_config.json":
{
	···
}
```
| dragonfly 配置文件 |  |
| --- | --- |
| proxy/addr | http://127.0.0.1:65001 |
| header | {"Accept": "*", "Host": "abc"} |
| filter | ["key", "sign"] |


### 具体实现
命名风格和具体实现参考官方实现的 checksum repoagent：
[https://github.com/triton-inference-server/checksum_repository_agent](https://github.com/triton-inference-server/checksum_repository_agent)
```cpp
/*
 *     Copyright 2023 The Dragonfly Authors
 *
 * Licensed under the Apache License, Version 2.0 (the "License");
 * you may not use this file except in compliance with the License.
 * You may obtain a copy of the License at
 *
 *      http://www.apache.org/licenses/LICENSE-2.0
 *
 * Unless required by applicable law or agreed to in writing, software
 * distributed under the License is distributed on an "AS IS" BASIS,
 * WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
 * See the License for the specific language governing permissions and
 * limitations under the License.
 */

#include "triton/core/tritonrepoagent.h"
#include "triton/core/tritonserver.h"
// 在此处导入 dragonfly 相关库

#include <openssl/md5.h>
#include <openssl/opensslv.h>

#include <algorithm>
#include <cctype>
#include <cstring>
#include <fstream>
#include <iomanip>
#include <memory>
#include <sstream>
#include <stdexcept>
#include <string>
#include <utility>
#include <vector>
#if OPENSSL_VERSION_NUMBER >= 0x30000000
#include <openssl/evp.h>
#endif

//
// Dragonfly Repository Agent that implements the TRITONREPOAGENT API.
//

namespace triton { namespace repoagent { namespace dragonfly {

namespace {
//
// ErrorException
//
// Exception thrown if error occurs while running DragonflyRepoAgent
//
struct ErrorException {
  ErrorException(TRITONSERVER_Error* err) : err_(err) {}
  TRITONSERVER_Error* err_;
};

#define THROW_IF_TRITON_ERROR(X)                                     \
  do {                                                               \
    TRITONSERVER_Error* tie_err__ = (X);                             \
    if (tie_err__ != nullptr) {                                      \
      throw triton::repoagent::dragonfly::ErrorException(tie_err__); \
    }                                                                \
  } while (false)


#define THROW_TRITON_ERROR(CODE, MSG)                                 \
  do {                                                                \
    TRITONSERVER_Error* tie_err__ = TRITONSERVER_ErrorNew(CODE, MSG); \
    throw triton::repoagent::dragonfly::ErrorException(tie_err__);    \
  } while (false)


#define RETURN_IF_ERROR(X)               \
  do {                                   \
    TRITONSERVER_Error* rie_err__ = (X); \
    if (rie_err__ != nullptr) {          \
      return rie_err__;                  \
    }                                    \
  } while (false)

class Dragonfly {
 public:
  // 此处写 dragonfly 相关函数与变量定义
  void
  LocalizeModelWithDragonfly(const std::string& config_path, const std::string& cred_path, const std::string& location, const std::string& temp_dir)
  {
      // TODO: 如果 d7y 不支持的对象存储需要抛错
      // HTTP Proxy 方式
      // proxy 地址 header（map） filter（数组）
  }

  template <class CredentialType>
  static void LoadCredential(const std::string& cred_path, CredentialType& credential)
  {
      // refer to Triton Core
  } 

  static void LoadDragonflyConfig(const std::string& config_path, DragonflyConfig& config)
  {
      
  }
};

}  // namespace

/////////////

extern "C" {

TRITONSERVER_Error*
TRITONREPOAGENT_ModelAction(
    TRITONREPOAGENT_Agent* agent, TRITONREPOAGENT_AgentModel* model,
    const TRITONREPOAGENT_ActionType action_type)
{
  switch (action_type) {
    case TRITONREPOAGENT_ACTION_LOAD: {

      // 通过 ModelRepositoryLocation 传入模型初始地址 location s3://
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

      // 通过环境变量传入 dragonfly 配置文件信息
      const char* config_path_c_str =
          std::getenv("TRITON_DRAGONFLY_CONFIG_PATH");
      std::string config_path;
      if (config_path_c_str != nullptr) {
        // Load from config file
        config_path = std::string(config_path_c_str);
      }

      // 通过 TRITONREPOAGENT_ModelRepositoryLocationAcquire
      // 函数获得一个本地暂存地址
      const char* temp_dir_cstr = nullptr;
      RETURN_IF_ERROR(TRITONREPOAGENT_ModelRepositoryLocationAcquire(
          agent, model, TRITONREPOAGENT_ARTIFACT_FILESYSTEM, &temp_dir_cstr));
      const std::string temp_dir(temp_dir_cstr);

      try {
        // 此处调用 dragonfly 类中的函数，传入 location、cred_path、config_path、temp_dir 
        // dragonfly 将目标模型下载至 temp_dir

        // 通过以下方式更新模型仓库地址
        RETURN_IF_ERROR(TRITONREPOAGENT_ModelRepositoryUpdate(
            agent, model, TRITONREPOAGENT_ARTIFACT_FILESYSTEM, temp_dir));
      }
      catch (const ErrorException& ee) {
        return ee.err_;
      }

      return nullptr;  // success
    }
    case TRITONREPOAGENT_ACTION_UNLOAD: {
      RETURN_IF_ERROR(TRITONREPOAGENT_ModelRepositoryLocationRelease(
          agent, model, temp_dir_cstr));

      return nullptr;
    }
    default:
      return nullptr;
  }

}  // extern "C"

}}}  // namespace triton::repoagent::dragonfly
```
