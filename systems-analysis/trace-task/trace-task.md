# Trace Task

## Overview

This design document proposes a new task type for the Dragonfly client called TraceTask.
nydusd records the chunk groups it reads on demand during one run and uploads the list to dfdaemon.
dfdaemon fetches the bytes it does not have locally, assembles them in access order into a single
local file, and serves it back on the next mount as one HTTP stream with `Range` support.
A cold start that used to issue hundreds or thousands of small range requests issues one.

## Goals

1. Add `TraceTask`, a local task built from the chunk groups accessed by a nydusd instance and served over HTTP with `Range`.
2. Reuse the existing storage, garbage collection and proxy patterns; keep `dragonfly-api` and the configuration unchanged.
3. Make the upload idempotent on an id the client can derive offline, so the download needs no lookup.
4. Provide client-side replication (two replicas, read fallback) in `dragonfly-sdk` for Rust and Go.

## Architecture

![Trace Task Workflow](./sequence-diagram.png)

### Modules

```text
dragonfly-client-storage/src
|-metadata.rs                 // TraceTask metadata and lifecycle operations
|-content.rs                  // DEFAULT_TRACE_TASK_DIR
|-content_linux.rs            // create/write/read/delete trace task content
|-content_macos.rs            // same for macOS
|-lib.rs                      // Storage wrappers

dragonfly-client/src
|-resource
|   |-mod.rs                  // Register trace_task
|   |-trace_task.rs           // TraceTask manager: upload, build, download; header and entry codec
|-proxy
|   |-mod.rs                  // Dispatch /v1/trace-tasks/ to the handler; shared helpers
|   |-trace_task.rs           // HTTP handler, request body types, status mapping
|   |-header.rs               // X-Dragonfly-Nydusd-ID
|-gc/mod.rs                   // Evict trace tasks by ttl and disk usage
|-bin/dfdaemon/main.rs        // Construct the manager, pass to Proxy and GC

dragonfly-client-metric/src
|-lib.rs                      // trace task metrics

dragonfly-sdk/client-request
|-rust/src/id_generator.rs    // trace_task_id, nydusd_id
|-rust/src/lib.rs             // put_trace_task, get_trace_task, decode_trace_task_entries
|-go/...                      // Go parity and consistency vectors
```

## Implementation

### API

nydusd sends requests directly to the dfdaemon proxy port (default `4001`).
Both endpoints require the `X-Dragonfly-Nydusd-ID` header, used only for tracing spans and logs.
The SDK generates it once per nydusd process the same way `IDGenerator::peer_id` is generated:

```rust
/// Generates the nydusd id.
pub fn nydusd_id(&self) -> String {
    format!("{}-{}-{}", self.ip, self.hostname, Uuid::new_v4())
}
```

#### ID

The trace task id is derived by the SDK so that the download needs no lookup:

```rust
/// Generates the trace task id by the digest of the bootstrap of the image.
pub fn trace_task_id(&self, bootstrap_digest: &str) -> String {
    hex::encode(Sha256::digest(format!("{bootstrap_digest}-trace-task")))
}
```

`bootstrap_digest` is the SHA256 of the bootstrap file passed to `nydus fuse --bootstrap`,
as 64 lowercase hex characters without the `sha256:` prefix. dfdaemon does not derive the id;
it only validates that the path segment matches `^[0-9a-f]{64}$`.

#### PUT /v1/trace-tasks/{id}

```http
PUT /v1/trace-tasks/3f0c…e9a1 HTTP/1.1
Content-Type: application/json
X-Dragonfly-Nydusd-ID: 10.0.0.8-node-1-7c1e…9f2b

{"chunk_groups":[
  {"blob_index":1,"chunk_group_index":0,
   "url":"https://registry.example.com/v2/app/blobs/sha256:aaaa…",
   "range":{"start":0,"length":1048576}},
  {"blob_index":1,"chunk_group_index":5,
   "url":"https://registry.example.com/v2/app/blobs/sha256:aaaa…",
   "range":{"start":5242880,"length":262144}},
  {"blob_index":2,"chunk_group_index":0,
   "url":"https://registry.example.com/v2/app/blobs/sha256:bbbb…",
   "range":{"start":0,"length":65536}}
]}
```

```rust
pub struct UploadTraceTaskRequest {
    pub chunk_groups: Vec<UploadTraceTaskChunkGroup>,
}

pub struct UploadTraceTaskChunkGroup {
    pub blob_index: u32,
    pub chunk_group_index: u32,
    pub url: String,
    pub range: Range,
}
```

Each item is one nydus chunk group. `blob_index` and `chunk_group_index` are the same pair nydus records
in its own `/trace` document and are copied into the entry table for replay. `url` and `range` locate the
compressed bytes of the group in the source blob; dfdaemon uses them only to download what is missing locally
and does not persist them.

The upload is idempotent on `id`: the same id is built once, and a repeated `PUT` returns the current state.
A later `PUT` for the same id with different chunk groups is ignored; the first upload wins until the task is evicted.

| Status | Condition                                                                                                     |
| ------ | ------------------------------------------------------------------------------------------------------------- |
| 202    | Created, or already building                                                                                  |
| 200    | Already finished                                                                                              |
| 400    | Invalid id or `X-Dragonfly-Nydusd-ID`; empty `chunk_groups`; url scheme not http/https; `range.length == 0`; malformed JSON |
| 413    | Body larger than 4 MiB or more than 8192 chunk groups                                                         |
| 507    | `has_enough_space(content_length)` is false                                                                   |

#### GET /v1/trace-tasks/{id}

```http
GET /v1/trace-tasks/3f0c…e9a1 HTTP/1.1
X-Dragonfly-Nydusd-ID: 10.0.0.8-node-1-7c1e…9f2b
Range: bytes=0-87

HTTP/1.1 206 Partial Content
Content-Type: application/octet-stream
Accept-Ranges: bytes
Content-Range: bytes 0-87/1376344
Content-Length: 88
X-Dragonfly-Server-IP: 10.0.0.12
```

| Status    | Condition                                                                 |
| --------- | ------------------------------------------------------------------------- |
| 200 / 206 | Finished; `Range` is parsed by `parse_range_header`, first range only    |
| 404       | Not found or still building                                               |
| 416       | Range not satisfiable, with `Content-Range: bytes */{content_length}`     |
| 400       | Invalid id or `X-Dragonfly-Nydusd-ID`                                     |
| 405       | Any method other than `PUT` and `GET`                                     |

A read failure in the middle of the stream closes the connection; there is no partial-success semantics.
`If-Range` and `multipart/byteranges` are not supported.

### Content Format

The file follows the nydus blob metadata style: a fixed-size header, a fixed-size entry table, then raw data.
All integers are little-endian. The entry table is placed before the data, because it is fully known at
upload time and a reader can consume the stream in one pass.

![Trace Task Content Format](./content-format.png)

For the three chunk groups above: `data_offset` is 88, the entry offsets are 0, 1048576 and 1310720,
and `content_length` is 1376344. A `Range: bytes=0-87` request returns the header and the entry table.

The file is a pure function of `chunk_groups`, so the two replicas are byte-identical. Entries carry no
digest: the source pieces were verified when they entered the local storage, and nydus verifies every
chunk group with CRC32C when it decodes it.

```rust
pub const TRACE_TASK_HEADER_SIZE: usize = 16;
pub const TRACE_TASK_ENTRY_SIZE: usize = 24;

pub struct TraceTaskHeader {
    pub entry_count: u32,
    pub entry_size: u32,
    pub data_offset: u64,
}

pub struct TraceTaskEntry {
    pub blob_index: u32,
    pub chunk_group_index: u32,
    pub offset: u64,
    pub length: u64,
}
```

### Storage

A trace task is one file at `{content_dir}/trace-tasks/{id}` without pieces. The metadata tracks the
lifecycle and the upload counters used by garbage collection; the entry table lives only in the file.

#### Metadata

```rust
/// The trace task built from the chunk groups accessed by a client,
/// served by the local dfdaemon only and never scheduled.
#[derive(Debug, Clone, Default, Serialize, Deserialize)]
pub struct TraceTask {
    pub id: String,
    pub content_length: u64,
    pub uploading_count: i64,
    pub uploaded_count: u64,
    pub updated_at: NaiveDateTime,
    pub created_at: NaiveDateTime,
    pub finished_at: Option<NaiveDateTime>,
}

impl DatabaseObject for TraceTask {
    const NAMESPACE: &'static str = "trace_task";
}

impl TraceTask {
    pub fn is_finished(&self) -> bool;
    pub fn is_uploading(&self) -> bool;
    pub fn is_expired(&self, ttl: Duration) -> bool;
}

impl Metadata {
    fn create_trace_task_started(id, content_length) -> TraceTask;
    fn create_trace_task_finished(id) -> TraceTask;
    fn upload_trace_task_started(id);
    fn upload_trace_task_finished(id);
    fn get_trace_task(id) -> Option<TraceTask>;
    fn get_trace_tasks() -> Vec<TraceTask>;
    fn delete_trace_task(id);
}
```

`Metadata::new` adds `TraceTask::NAMESPACE` to the point-lookup column families. RocksDB is opened with
`create_missing_column_families`, so existing databases need no migration. A failed build deletes the row,
so there is no `failed_at`.

#### Content

```rust
pub const DEFAULT_TRACE_TASK_DIR: &str = "trace-tasks";

async fn create_trace_task(id, content_length) -> PathBuf;    // fallocate
async fn write_trace_task(id, offset, length, reader);        // io::write_range
async fn read_trace_task(id, range) -> RangeReader;           // fd_cache + fadvise_willneed
async fn delete_trace_task(id);
```

#### Storage Wrappers

```rust
async fn create_trace_task_started(id, content_length) -> TraceTask;
async fn write_trace_task(id, offset, length, reader);
fn create_trace_task_finished(id) -> TraceTask;
async fn create_trace_task_failed(id);                        // = delete
async fn upload_trace_task(id, range) -> (TraceTask, RangeReader);
fn get_trace_task(id) -> Option<TraceTask>;
fn get_trace_tasks() -> Vec<TraceTask>;
async fn delete_trace_task(id);
```

`upload_trace_task` follows `upload_piece`: it increments `uploading_count` and refreshes `updated_at`
when the reader is created, and decrements the counter when the reader is dropped.

### Resource

![Trace Task Lifecycle](./state-diagram.png)

```rust
// dragonfly-client/src/resource/trace_task.rs
pub struct TraceTask {
    config: Arc<Config>,
    storage: Arc<Storage>,
    task: Arc<Task>,
    create: tokio::sync::Mutex<()>,   // serializes get -> create_started
}

pub enum UploadTraceTaskState { Accepted, Exists }

async fn upload(id, request, dynconfig, remote_ip) -> UploadTraceTaskState;
async fn download(id, range) -> Option<(TraceTask, RangeReader)>;
async fn build(id, chunk_groups, dynconfig, remote_ip);
```

#### Upload

1. Validate the request, encode the header and the entry table, compute `content_length`.
2. Take the `create` lock and `get_trace_task(id)`: finished returns `Exists`; building returns `Accepted`
   without a second build.
3. `has_enough_space(content_length)`, `create_trace_task_started`, release the lock.
4. `tokio::spawn(build)` and return `Accepted`.

The synchronous part performs no network I/O, so the `PUT` returns immediately with `202`.

A building row is always owned by a live `build` task: it ends in `create_trace_task_finished` or
`create_trace_task_failed`. Rows left unfinished by a crash are deleted when the manager starts, before the
proxy accepts requests, so no timeout is needed and two builds never write the same file.

#### Build

1. `write_trace_task(id, 0, data_offset, header || entry table)`.
2. Build one `DownloadTaskRequest` per chunk group with the same fields as a proxied `GET`:

   | Field            | Value                                                   |
   | ---------------- | ------------------------------------------------------- |
   | `range`          | `Some(range)`                                           |
   | `request_header` | empty                                                   |
   | `rule`           | `find_matching_rule` match, otherwise `Rule::default()` |
   | `prefetch`       | `false`                                                 |
   | `priority`       | `header::get_priority` on the `PUT` headers             |

   The task id is derived by `task::download` with the existing rules, so it matches the task that was
   written when nydusd read the range through the proxy, and local pieces are found without a scheduler
   round trip. Because `request_header` is empty, a chunk group that is not local and whose origin requires
   credentials fails the build.
3. `stream::iter(requests).map(task::download).buffered(concurrent_piece_count)`: up to
   `download.concurrent_piece_count` sessions are opened concurrently while the output stays in order.
   `Task::download` serves finished local pieces first and fetches the rest from peers or the origin.
4. Consume each session with `write_pieces_in_order(task, out_stream, started, task_id, sink)`, extracted
   from the ordered piece loop in `proxy_via_dfdaemon`. The sink is
   `write_trace_task(id, data_offset + offset, len, reader)`.
5. Verify that every chunk group wrote exactly `range.length` bytes. Any failure calls
   `create_trace_task_failed`, which deletes the row and the file; success calls `create_trace_task_finished`.

#### Download

1. `get_trace_task(id)`; anything but finished returns `404`.
2. If `Range` is present, `parse_range_header(value, content_length)`; failure returns `416`.
3. `storage.upload_trace_task(id, range)` returns the `RangeReader`.
4. Write the `200` or `206` headers and stream the reader through `send_reader(reader, body_tx)`,
   shared with the proxy send loop.

### Proxy

```rust
// proxy/mod.rs::handler, after rate limiting and before the registry mirror dispatch
if request.uri().host().is_none()
    && request.uri().path().starts_with(trace_task::PATH_PREFIX)
{
    return trace_task::handler(config, trace_task, dynconfig, request, remote_ip).await;
}

// proxy/trace_task.rs
pub const PATH_PREFIX: &str = "/v1/trace-tasks/";
const MAX_CHUNK_GROUPS: usize = 8192;
const MAX_BODY_LENGTH: usize = 4 * 1024 * 1024;
```

The body is read through `http_body_util::Limited` so oversized uploads are rejected before parsing.
`X-Dragonfly-Nydusd-ID` is added to `proxy/header.rs` next to the other `X-Dragonfly-*` headers and is
recorded as a span field; it is not a metrics label.

### Garbage Collection

| Method                           | Rule                                                                                                          |
| -------------------------------- | ------------------------------------------------------------------------------------------------------------- |
| `evict_trace_task_by_ttl`        | `is_expired(gc.policy.task_ttl)` and `!is_uploading()`; reuses the task ttl, no new option                    |
| `evict_trace_task_by_disk_usage` | Above the high watermark, evict finished tasks by ascending `updated_at` down to the low watermark, skipping uploading tasks; runs before task eviction because trace tasks are derived data |
| Startup                          | Unfinished rows and their files are deleted when the manager starts                                            |

### Metrics

| Metric                                            | Type      | Labels                                        |
| ------------------------------------------------- | --------- | --------------------------------------------- |
| `dragonfly_client_trace_task_upload_total`        | counter   | `state` = started / finished / failed         |
| `dragonfly_client_trace_task_download_total`      | counter   | `state` = started / finished / failed, `hit`  |
| `dragonfly_client_trace_task_build_duration_seconds` | histogram | none                                       |

### SDK

```rust
pub struct PutTraceTaskRequest {
    pub id: String,
    pub nydusd_id: String,
    pub chunk_groups: Vec<UploadTraceTaskChunkGroup>,
}

pub struct GetTraceTaskRequest {
    pub id: String,
    pub nydusd_id: String,
    pub range: Option<Range>,
}

impl ProxyWithEndpoints {
    /// Uploads the trace task to two replicas selected by the hash ring,
    /// succeeds when any replica returns 200 or 202.
    pub async fn put_trace_task(&self, request: PutTraceTaskRequest) -> Result<()>;

    /// Downloads the trace task from the replicas in ring order, falls back
    /// on 404 and returns None when every replica misses.
    pub async fn get_trace_task(&self, request: GetTraceTaskRequest) -> Result<Option<GetResponse>>;
}

/// Decodes the header and the entry table read from the start of the content.
pub fn decode_trace_task_entries(bytes: &[u8]) -> Result<Vec<TraceTaskEntry>>;
```

Replicas are chosen with `VNodeHashRing::get_with_replicas(&id, 2)`, so `PUT` and `GET` for one id land on
the same two endpoints. All `Range` reads of one replay go to the same replica. The Go module implements
`TraceTaskID`, `NydusdID`, `PutTraceTask`, `GetTraceTask` and `DecodeTraceTaskEntries` with shared
consistency vectors for the id and the codec.

### Nydus

1. After mount, record `(blob_index, chunk_group_index, url, range)` for every chunk group read on demand,
   in first-access order.
2. Call `put_trace_task` once, when no on-demand read has happened for 10 seconds or when 5 minutes have
   elapsed since mount, whichever comes first.
3. On the next mount, `GET` the header and the entry table. Skip the groups already present in the blob
   cache, fetch the rest with as few `Range` requests as possible or stream the whole file, decode and
   CRC32C-verify each group, write it into the source blob cache and mark it ready. A `404` or a short read
   falls back to on-demand reads.
