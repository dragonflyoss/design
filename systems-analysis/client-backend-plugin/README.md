# plugin

### Introduction

This document describes the use of a plugin for the Dragonfly client backend. It instructs developers to use the plugin to enable Dragonfly to support additional backends.

### Usage

The backend plugin is a dynamic link library, named in the format `lib{plugin-name}.so`.

**1. Put the plugin into the plugin directory**

After the plugin is built, it needs to be put into the `/usr/local/lib/dragonfly/plugins/dfdaemon/backend` path by default. This can be changed in the [dfdaemon config](https://d7y.io/docs/next/reference/configuration/client/dfdaemon/). `dfdaemon` does not support hot reloading of plugins.

**2. Loading Plugins**

A successful plugin load will produce the following log messages:

```
INFO  load [<plugin-name>] plugin backend
```

**3. Use the plugin to download files**

After the plugin is successfully loaded, you can use dfget to download the file:

```shell
dfget <plugin-name>://<host>:<port>/<path> --output /tmp/file.txt
```

### Details

Before you begin development, refer to the [plugin examples](https://github.com/dragonflyoss/client/tree/main/dragon-client-backend/examples/plugin) for guidance.

The backend plugin requires the implementation of two main interfaces: `head` and `get`.

`head` is used to get metadata of the task.
- For single file downloads, this refers to retrieving metadata such as the file size.

- For directory downloads, it retrieves information about all files in the current directory and its subdirectories, including their file paths and sizes.

```rust
async fn head(&self, request: HeadRequest) -> Result<HeadResponse> {
    ...
}
```

`get` is used to download the content of piece.
```rust
async fn get(&self, request: GetRequest) -> Result<GetResponse<Body>> {
    ...
}
```

### Workflow

**Plugin Registration**

Upon startup, `dfdaemon` scans the plugin path.

If plugins are found, they are registered. Otherwise, this step is skipped.

![](./register-plugin.jpg)

**Download File**

During file download, `dfdaemon` first heads the file metadata through plugin. The file is split into multiple pieces based on its size, with each piece downloaded independently via the plugin's get request.

![](./download-file.jpg)

**Download Directory**

During directory download, `dfdaemon` first heads the metadatas through plugin for all files in the directory, then each file download creates a separate download request.

![](./download-directory.jpg)
