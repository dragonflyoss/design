# plugin

### Introduction

This document describes the use of a plugin for the dragonfly client backend. It instructs developers to use the plugin to enable Dragonfly to support additional backends, such as dfs, without having to break into Dragonlfy to make code changes.

### Usage

The plugin for backend exists as a lib{plugin-name}.so dynamic link library. Here are the steps to use it:

**1. plugin storage in the specified path**

After the plugin is built, it needs to be stored in the `/usr/local/lib/dragonfly/plugins/dfdaemon/backend` directory by default. This can be changed in [dfdaemon config](https://d7y.io/docs/next/reference/configuration/client/dfdaemon/).

Dfdaemon will only load the plugin at startup, so dfdaemon needs to be restarted after the plugin is put in.

**2. dfdaemon loading plugin**

When dfdaemon starts, the following logs are included in the dfdaemon startup log, which means that the plugin was loaded successfully.

```
INFO  load [<plugin-name>] plugin backend
```

**3. Use the backend of the plugin to download files**

After the plugin is successfully loaded, you can download the file via dfget:

```shell
dfget <plugin-name>://<host>:<port>/<path> --output /tmp/file.txt
```

### Details

Before development, please refer to [Dragonfly Client](https://github.com/dragonflyoss/client/tree/main) for the [dragonfly-client-backend/examples/plugin](https://github.com/dragonflyoss/client/tree/main/dragonfly-client-backend/examples/plugin) module.

There is no need to change the code of Dragonfly client during development. You need to implement two interfaces of Backend, `head` and `get`.

`head` is used to get metadata information about the task.

- For single file download, it means getting metadata such as file size.

- For directory download, it will get information about files in the current directory and all subdirectories, including file paths and file sizes.

```rust
async fn head(&self, request: HeadRequest) -> Result<HeadResponse> {
    ...
}
```

`get` is used to download the data content of the corresponding piece.

```rust
async fn get(&self, request: GetRequest) -> Result<GetResponse<Body>> {
    ...
}
```

### Workflow

**Plugin Registration**

Upon startup, dfdaemon scans the designated plugin path. If plugins are found, they are registered sequentially; otherwise, this step is skipped.

![](./register-plugin.jpg)

**Download File**

During file download, DFCaemon first obtains the file size through plugin. The file is then divided into multiple pieces based on its size, and each piece establishes an independent download task through the plugin.

![](./download-file.jpg)

**Download Directory**

When downloading a directory, dfget first retrieves metadata for all files within that directory, including their path and size. If the user specifies the include-files parameter, dfget filters for the files. Finally, it iterates through each remaining file to establish individual download requests.

![](./download-directory.jpg)
