# Streaming gRPC Support for dfcache Client in Dragonfly

## Overview

This design document proposes enhancing Dragonfly's `dfcache` client to support streaming gRPC interfaces for persistent cache tasks in Kubernetes environments using Unix Domain Sockets (UDS). The enhancements aim to improve the efficiency of large file transfers and user experience by integrating streaming interfaces, optimizing UDS connections, and ensuring Kubernetes compatibility.

## Motivation

- **Performance**: Streaming gRPC interfaces reduce I/O overhead and enable efficient large file transfers compared to non-streaming methods.
- **Flexibility**: Streaming support enhances interactivity with progress tracking and task control.
- **Compatibility**: UDS and Kubernetes optimizations address file path and permission challenges in containerized environments.
- **User Experience**: Improved progress indicators and error handling enhance usability.

## Design Goals

1. Add streaming gRPC support (`WritePersistentCacheTask` and `ReadPersistentCacheTask`) while maintaining compatibility with existing `dfcache` functionality.
2. Preserve the simplicity and maintainability of the `dfcache` client architecture.
3. Optimize UDS connections for performance and reliability in Kubernetes.
4. Enhance file transfer efficiency with piece-based streaming and robust error recovery.

## Architecture

### Current Architecture

```
dfcache -> {
    main.rs: CLI entry with clap parsing
    import.rs: ImportCommand with UploadPersistentCacheTaskRequest
    export.rs: ExportCommand with DownloadPersistentCacheTaskRequest
    stat.rs: StatCommand
}
DfdaemonDownloadClient -> gRPC (non-streaming)
```

### Proposed Architecture

```
dfcache -> {
    main.rs: CLI entry with clap parsing
    import.rs: ImportCommand with WritePersistentCacheTask (streaming)
    export.rs: ExportCommand with ReadPersistentCacheTask (streaming)
    stat.rs: StatCommand
}
DfdaemonDownloadClient -> gRPC (streaming and non-streaming)
```

### Module Structure

#### Modified Modules

- `dragonfly-client/src/grpc/dfdaemon_download.rs`: Add streaming gRPC methods (`write_persistent_cache_task`, `read_persistent_cache_task`).
- `dragonfly-client/src/cli/import.rs`: Refactor `ImportCommand` for streaming uploads.
- `dragonfly-client/src/cli/export.rs`: Refactor `ExportCommand` for streaming downloads.
- `dragonfly-client/src/storage/content.rs`: Enhance file read/write with streaming optimizations.
- Configuration file: Add streaming-specific options.

## Technical Implementation

### 1. Core Interfaces

#### Streaming gRPC Interfaces

```protobuf
// Write persistent cache task to P2P network
rpc WritePersistentCacheTask(stream WritePersistentCacheTaskRequest) returns (WritePersistentCacheTaskResponse);

message WritePersistentCacheTaskRequest {
  oneof response {
    WritePersistentCacheTaskStartedRequest write_persistent_cache_task_started_request = 1;
    WritePersistentCacheTaskFinishedRequest write_persistent_cache_task_finished_request = 2;
    WriteChunkRequest write_chunk_request = 3;
  }
}

message WritePersistentCacheTaskStartedRequest {
  uint64 persistent_replica_count = 1;
  optional string tag = 2;
  optional string application = 3;
  google.protobuf.Duration ttl = 4;
  uint64 content_length = 5;
}

message WritePersistentCacheTaskFinishedRequest {}
message WritePersistentCacheTaskResponse { string task_id = 1; }
message WriteChunkRequest { bytes content = 1; }

// Read persistent cache task from P2P network
rpc ReadPersistentCacheTask(ReadPersistentCacheTaskRequest) returns (stream ReadPersistentCacheTaskResponse);

message ReadPersistentCacheTaskRequest { string task_id = 1; }
message ReadPersistentCacheTaskResponse {
  oneof response {
    ReadPersistentCacheTaskFinishedResponse read_persistent_cache_task_finished_response = 1;
    ReadChunkResponse read_chunk_response = 2;
  }
}
message ReadPersistentCacheTaskFinishedResponse {}
message ReadChunkResponse { bytes content = 1; }
```

#### Client Methods

```rust
impl DfdaemonDownloadClient {
    pub async fn write_persistent_cache_task(
        &self,
        request: impl tonic::IntoStreamingRequest<Message = WritePersistentCacheTaskRequest>,
    ) -> ClientResult<WritePersistentCacheTaskResponse> {
        let response = self.client.write_persistent_cache_task(request).await?;
        Ok(response.into_inner())
    }

    pub async fn read_persistent_cache_task(
        &self,
        request: ReadPersistentCacheTaskRequest,
    ) -> ClientResult<tonic::Response<tonic::codec::Streaming<ReadPersistentCacheTaskResponse>>> {
        let request = tonic::Request::new(request);
        let response = self.client.read_persistent_cache_task(request).await?;
        Ok(response)
    }
}
```

### 2. UDS Connection Optimization

#### Connection Pooling

```rust
pub struct UDSConnectionPool {
    config: ConnectionPoolConfig,
    idle_connections: HashMap<PathBuf, VecDeque<Arc<UDSConnection>>>,
    active_connections: AtomicUsize,
    metrics: ConnectionPoolMetrics,
    _cleanup_task: JoinHandle<()>,
}
```

#### Connection Lifecycle Management

- **Create**: Establish new UDS connections when none are available, recording creation time.
- **Borrow**: Validate connection health before use, discarding unhealthy connections.
- **Return**: Return connections to the pool, updating last-used time.
- **Cleanup**: Periodically remove idle connections exceeding max idle time.

#### Auto-Reconnect Mechanism

```rust
impl UDSConnectionPool {
    pub async fn get_connection(&self, path: &Path) -> Result<Arc<UDSConnection>> {
        if let Some(conn) = self.try_get_idle_connection(path) {
            if conn.is_healthy().await {
                return Ok(conn);
            }
        }

        let mut retry_count = 0;
        let max_retries = 3;
        let mut last_error = None;

        while retry_count < max_retries {
            match self.create_new_connection(path).await {
                Ok(conn) => return Ok(conn),
                Err(err) => {
                    retry_count += 1;
                    last_error = Some(err);
                    tokio::time::sleep(Duration::from_millis(100 * 2u64.pow(retry_count))).await;
                }
            }
        }
        Err(last_error.unwrap_or_else(|| Error::ConnectionFailed))
    }
}
```

#### Zero-Copy Implementation

```rust
pub struct ZeroCopyTransfer {
    fd: RawFd,
    file_size: u64,
}

impl ZeroCopyTransfer {
    #[cfg(target_os = "linux")]
    pub fn send_to_socket(&self, socket_fd: RawFd, offset: u64, length: u64) -> Result<usize> {
        unsafe { libc::sendfile(socket_fd, self.fd, &mut offset, length) }
    }

    pub fn send_fd_to_uds(&self, uds_socket: &UnixStream) -> Result<()> {
        // Implement SCM_RIGHTS for file descriptor passing
    }
}
```

### 3. File Read/Write Enhancements

#### Piece-based Streaming

```protobuf
message Piece {
    uint32 number = 1;
    string parent_id = 2;
    uint64 offset = 3;
    uint64 length = 4;
    common.v2.PieceDigest digest = 5;
    bytes content = 6;
}
```

#### Optimized File Reading

```rust
impl Content {
    pub async fn read_piece(
        &self,
        task_id: &str,
        offset: u64,
        length: u64,
        range: Option<Range>,
    ) -> Result<impl AsyncRead> {
        let task_path = self.get_task_path(task_id);
        let (target_offset, target_length) = calculate_piece_range(offset, length, range);
        let f = File::open(task_path.as_path()).await?;
        let mut f_reader = BufReader::with_capacity(self.config.storage.read_buffer_size, f);
        f_reader.seek(SeekFrom::Start(target_offset)).await?;
        Ok(f_reader.take(target_length))
    }
}
```

#### Optimized File Writing with Zero-Copy

```rust
impl Content {
    pub async fn write_piece<R: AsyncRead + Unpin + ?Sized>(
        &self,
        task_id: &str,
        offset: u64,
        reader: &mut R,
    ) -> Result<WritePieceResponse> {
        let task_path = self.get_task_path(task_id);
        let mut f = OpenOptions::new().write(true).open(task_path.as_path()).await?;
        f.seek(SeekFrom::Start(offset)).await?;
        let reader = BufReader::with_capacity(self.config.storage.write_buffer_size, reader);
        let mut writer = BufWriter::with_capacity(self.config.storage.write_buffer_size, f);
        let mut hasher = crc32fast::Hasher::new();
        let mut tee = InspectReader::new(reader, |bytes| hasher.update(bytes));
        let length = io::copy(&mut tee, &mut writer).await?;
        writer.flush().await?;
        Ok(WritePieceResponse {
            length,
            hash: hasher.finalize().to_string(),
        })
    }
}
```

#### Buffer Pooling

```rust
pub struct BufferPool {
    available_buffers: Mutex<Vec<Box<[u8]>>>,
    buffer_size: usize,
    max_buffers: usize,
    allocated_count: AtomicUsize,
}
```

### 4. Error Handling and Recovery

#### Streaming Retry Mechanism

- Implement retry logic for UDS and gRPC streaming failures with exponential backoff (max 3 retries).
- Perform health checks before reusing connections.

#### Piece-level Recovery

- Track completed pieces for breakpoint resuming.
- Validate piece integrity using digests.
- Retransmit failed pieces upon reconnection.

#### Monitoring Enhancements

- Monitor transfer rates, latency, and errors.
- Adjust transfer rates or reconnect on anomalies.
- Provide detailed logs for debugging.

### 5. Configuration

Add streaming and UDS configuration to `dfcache.yaml`:

```yaml
client:
  streaming:
    buffer_size: 4194304
    max_connections: 100
    idle_timeout: 300s
  uds:
    socket_path: /var/run/dragonfly.sock
```

## Testing Strategy

1. **Unit Tests**: Test new gRPC interfaces and client functionality (85%+ coverage).
2. **Integration Tests**: Verify end-to-end streaming in Kubernetes environments.
3. **Performance Tests**: Benchmark streaming vs. non-streaming performance.
4. **Stress Tests**: Test high-concurrency and network instability scenarios.

## Expected Outcomes

1. **Functionality**: Full streaming support for persistent cache tasks.
2. **Performance**: more improvement in large file transfer efficiency.
3. **Reliability**: Robust error handling and recovery for streaming operations.
4. **Documentation**: Comprehensive guides for developers and users.

## Compatibility

- **Backward Compatibility**: Existing non-streaming functionality remains unchanged.
- **Configuration**: Streaming options with sensible defaults.
- **Migration**: Users can enable streaming via configuration without code changes.

## Future Considerations

- **Dynamic Interface Selection**: Automatically choose streaming or non-streaming based on file size.
- **Advanced Task Control**: Add pause/resume capabilities for transfers.
- **Extended Monitoring**: Integrate with observability tools for detailed metrics
