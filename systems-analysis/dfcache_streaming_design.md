# Design Document: Dragonfly dfcache Streaming Upload/Download with Container Path Mapping

## Overview

This design document proposes enhancing the Dragonfly dfcache client to support streaming-based persistent cache operations with container path mapping capabilities. The enhancement will leverage dfdaemon's new gRPC streaming interfaces (`WritePersistentCacheTask` and `ReadPersistentCacheTask`) to enable efficient large file transfers while addressing the path mapping challenges commonly encountered in Kubernetes containerized environments.

The project focuses on extending the existing Rust dfcache client within the `dragonflyoss/client` codebase, maintaining full compatibility with Dragonfly's design principles and existing components while introducing advanced streaming capabilities and seamless container-to-host path translation.

## Motivation

  * **Path Mapping Challenges**: In Kubernetes environments, dfcache clients running inside containers face significant challenges due to path differences between container internal paths and host paths where dfdaemon operates. This creates access permission issues and prevents effective file import/export operations.

  * **Large File Transfer Inefficiency**: Current dfcache implementation relies on non-streaming `UploadPersistentCacheTaskRequest` interface, which lacks efficiency for large file uploads and provides limited progress control and fine-grained task management capabilities.

  * **Enhanced User Experience Requirements**: Users require better progress indication, more comprehensive error handling, and improved task control functionality, especially when dealing with large files in production container environments.

  * **Streaming Interface Adaptation**: dfdaemon's introduction of new streaming gRPC interfaces (`WritePersistentCacheTask` and `ReadPersistentCacheTask`) presents an opportunity to significantly improve file transfer efficiency and user experience while maintaining system stability.

## Goals

1. **Streaming Interface Adaptation**: Enable dfcache client to fully utilize dfdaemon's new streaming gRPC interfaces (`WritePersistentCacheTask` and `ReadPersistentCacheTask`) for efficient large file import/export operations with enhanced progress control and task management capabilities.

2. **Container Path Mapping Implementation**: Develop a robust path mapping mechanism that seamlessly translates container internal paths to host paths, resolving file access and permission challenges in Kubernetes environments.

3. **User Experience Enhancement**: Provide comprehensive progress indication, improved error handling, and enhanced task control functionality to deliver a superior user experience, particularly for large file operations in production environments.

4. **Code Quality and Maintainability**: Ensure all enhancements maintain the highest code quality standards while preserving the simplicity and maintainability of the dfcache client, with full backward compatibility for existing functionality.

5. **Performance Optimization**: Achieve optimal file transfer efficiency through streaming operations while maintaining minimal performance overhead, ensuring the solution scales effectively for large files and high-concurrency scenarios.

## Architecture

The proposed architecture introduces streaming capabilities and path mapping components within the existing dfcache client structure, while maintaining seamless integration with Dragonfly's existing piece management and gRPC communication systems.

### Data Flow Architecture

**Upload Flow (Import):**
```
Container File -> PathMapper -> Host File -> PieceCollector -> StreamingUpload -> dfdaemon -> P2P Network
                     ↓                          ↓                    ↓
              Path Translation        Piece Chunking        gRPC Streaming
```

**Download Flow (Export):**
```
P2P Network -> dfdaemon -> StreamingDownload -> PieceAssembler -> Host File -> PathMapper -> Container File
                              ↓                      ↓               ↓
                        gRPC Streaming        Piece Reconstruction  Path Translation
```

### System Integration

The architecture integrates with existing Dragonfly components:
- **Piece Management**: Utilizes existing `piece.rs` for chunking and ID generation
- **gRPC Client**: Extends `DfdaemonDownloadClient` with streaming methods
- **Progress Tracking**: Leverages existing progress bar and metrics collection
- **ID Generation**: Reuses `id_generator` for task identification

### Modules

```
dragonfly-client/src/bin/dfcache/
├── import.rs           # Enhanced with --stream-mode and --path-mapping
├── export.rs           # Enhanced with --stream-mode and --path-mapping  
└── path_mapper.rs      # New path mapping component

dragonfly-client/src/grpc/
└── dfdaemon_download.rs # Extended with streaming methods
```

### Component Details

* **PathMapper**: Handles bidirectional container-to-host path translation with configurable mapping rules
* **StreamingUpload**: Implements `WritePersistentCacheTask` with concurrent piece upload and progress tracking
* **StreamingDownload**: Implements `ReadPersistentCacheTask` with ordered piece assembly and progress reporting
* **Enhanced Commands**: Extended import/export commands with streaming mode and path mapping options

## Implementation

### Path Mapper Component

```rust
/// Path mapping component for container-to-host path translation
pub struct PathMapper {
    mappings: Vec<(PathBuf, PathBuf)>,
}

impl PathMapper {
    pub fn new(mappings: Vec<(PathBuf, PathBuf)>) -> Self {
        Self { mappings }
    }
    
    /// Map container path to host path
    pub fn map_container_path_to_host(&self, container_path: &Path) -> Option<PathBuf> {
        for (container_prefix, host_prefix) in &self.mappings {
            if let Ok(rel_path) = container_path.strip_prefix(container_prefix) {
                return Some(host_prefix.join(rel_path));
            }
        }
        None
    }
    
    /// Map host path to container path  
    pub fn map_host_path_to_container(&self, host_path: &Path) -> Option<PathBuf> {
        for (container_prefix, host_prefix) in &self.mappings {
            if let Ok(rel_path) = host_path.strip_prefix(host_prefix) {
                return Some(container_prefix.join(rel_path));
            }
        }
        None
    }
}

/// Parse path mapping string: "container_path:host_path[,...]"
pub fn parse_path_mappings(mapping_str: &str) -> Result<Vec<(PathBuf, PathBuf)>> {
    let mut mappings = Vec::new();
    for mapping in mapping_str.split(',') {
        let parts: Vec<&str> = mapping.split(':').collect();
        if parts.len() != 2 {
            return Err(Error::ValidationError(format!(
                "Invalid path mapping format: {}", mapping
            )));
        }
        mappings.push((PathBuf::from(parts[0]), PathBuf::from(parts[1])));
    }
    Ok(mappings)
}
```

### Enhanced Command Interface

```rust
/// Extended ImportCommand with streaming and path mapping support
#[derive(Debug, Clone, Parser)]
pub struct ImportCommand {
    // Existing fields...
    
    #[arg(long = "stream-mode", help = "Use streaming mode for large file transfers")]
    stream_mode: bool,
    
    #[arg(long = "path-mapping", help = "Container to host path mapping: container:host[,...]")]
    path_mapping: Option<String>,
}

impl ImportCommand {
    async fn run(&self, client: DfdaemonDownloadClient) -> Result<()> {
        // Handle path mapping
        let host_path = if let Some(mapping_str) = &self.path_mapping {
            let mappings = parse_path_mappings(mapping_str)?;
            let mapper = PathMapper::new(mappings);
            mapper.map_container_path_to_host(&self.path)
                .unwrap_or_else(|| self.path.clone())
        } else {
            self.path.clone()
        };
        
        if self.stream_mode {
            self.run_streaming_mode(client, &host_path).await
        } else {
            self.run_legacy_mode(client, &host_path).await
        }
    }
    
    async fn run_streaming_mode(&self, client: DfdaemonDownloadClient, path: &Path) -> Result<()> {
        let file_size = tokio::fs::metadata(path).await?.len();
        let piece_length = self.calculate_piece_length(file_size);
        
        // Create streaming channel
        let (tx, rx) = mpsc::channel(8);
        
        // Send task start request
        tx.send(WritePersistentCacheTaskRequest {
            response: Some(WritePersistentCacheTaskStartedRequest {
                persistent_replica_count: self.persistent_replica_count,
                content_length: file_size,
                piece_length,
                // ... other fields
            }.into())
        }).await?;
        
        // Stream file as pieces
        stream_file_as_pieces(path, tx.clone(), piece_length).await?;
        
        // Send completion request
        tx.send(WritePersistentCacheTaskRequest {
            response: Some(WritePersistentCacheTaskFinishedRequest {}.into())
        }).await?;
        
        // Execute streaming request
        let response = client.write_persistent_cache_task(ReceiverStream::new(rx)).await?;
        info!("Task completed: {}", response.task_id);
        
        Ok(())
    }
}
```

### Streaming Upload Implementation

```rust
/// Stream file as pieces with concurrent upload
async fn stream_file_as_pieces(
    file_path: &Path,
    tx: mpsc::Sender<WritePersistentCacheTaskRequest>,
    piece_length: u64,
) -> Result<()> {
    let file = Arc::new(tokio::fs::File::open(file_path).await?);
    let file_size = file.metadata().await?.len();
    let piece_count = (file_size + piece_length - 1) / piece_length;
    
    // Use semaphore to control concurrency
    let semaphore = Arc::new(Semaphore::new(4));
    let mut join_set = JoinSet::new();
    
    for piece_number in 0..piece_count {
        let permit = semaphore.clone().acquire_owned().await?;
        let file = file.clone();
        let tx = tx.clone();
        let offset = piece_number * piece_length;
        let length = std::cmp::min(piece_length, file_size - offset);
        
        join_set.spawn(async move {
            let mut buf = vec![0u8; length as usize];
            let mut file_reader = &*file;
            file_reader.seek(SeekFrom::Start(offset)).await?;
            file_reader.read_exact(&mut buf).await?;
            
            tx.send(WritePersistentCacheTaskRequest {
                response: Some(WritePieceRequest {
                    content: buf,
                }.into())
            }).await?;
            
            drop(permit);
            Ok::<_, Error>(())
        });
    }
    
    // Wait for all pieces to complete
    while let Some(result) = join_set.join_next().await {
        result??;
    }
    
    Ok(())
}
```

### Streaming Download Implementation

```rust
/// Enhanced ExportCommand with streaming download
impl ExportCommand {
    async fn run_streaming_mode(&self, client: DfdaemonDownloadClient, output_path: &Path) -> Result<()> {
        let request = ReadPersistentCacheTaskRequest {
            task_id: self.id.clone(),
        };
        
        let mut stream = client.read_persistent_cache_task(request).await?.into_inner();
        let file = OpenOptions::new()
            .create(true)
            .truncate(true)
            .write(true)
            .open(output_path)
            .await?;
        let mut file_writer = BufWriter::new(file);
        
        while let Some(response) = stream.message().await? {
            match response.response {
                Some(ReadPersistentCacheTaskStartedResponse { content_length, .. }) => {
                    // Initialize progress bar with content length
                    info!("Starting download, size: {} bytes", content_length);
                }
                Some(ReadPieceResponse { content, .. }) => {
                    file_writer.write_all(&content).await?;
                }
                Some(ReadPersistentCacheTaskFinishedResponse { .. }) => {
                    file_writer.flush().await?;
                    info!("Download completed");
                    break;
                }
                None => {}
            }
        }
        
        Ok(())
    }
}
```

### gRPC Client Extensions

```rust
impl DfdaemonDownloadClient {
    /// Streaming upload method
    pub async fn write_persistent_cache_task(
        &self,
        request: impl tonic::IntoStreamingRequest<Message = WritePersistentCacheTaskRequest>,
    ) -> ClientResult<WritePersistentCacheTaskResponse> {
        let response = self.client.write_persistent_cache_task(request).await?;
        Ok(response.into_inner())
    }
    
    /// Streaming download method
    pub async fn read_persistent_cache_task(
        &self,
        request: ReadPersistentCacheTaskRequest,
    ) -> ClientResult<tonic::Response<tonic::codec::Streaming<ReadPersistentCacheTaskResponse>>> {
        let response = self.client.read_persistent_cache_task(request).await?;
        Ok(response)
    }
}
```

### Usage Examples

**Basic Streaming Import:**
```bash
# Import large file using streaming mode
dfcache import --stream-mode --piece-length=32mib /path/to/large/dataset.tar.gz

# Import with custom replica count and TTL
dfcache import --stream-mode --persistent-replica-count=3 --ttl=24h /path/to/model.bin
```

**Container Environment with Path Mapping:**
```bash
# Single path mapping
dfcache import --stream-mode --path-mapping="/data:/var/lib/kubelet/pods/pod-123/volumes/pvc-abc" /data/large/file.bin

# Multiple path mappings
dfcache import --stream-mode --path-mapping="/data:/host/data,/tmp:/host/tmp" /data/cache/artifacts.zip

# Export with path mapping
dfcache export --stream-mode --path-mapping="/output:/host/output" --output=/output/result.bin <task-id>
```

**Production Deployment Example:**
```yaml
# Kubernetes Pod with dfcache streaming
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: dfcache-client
    image: dragonflyoss/client:latest
    command: ["/bin/dfcache"]
    args: ["import", "--stream-mode", "--path-mapping=/data:/var/lib/kubelet/pods/$(POD_NAME)/volumes/pvc-data", "/data/large-dataset.tar.gz"]
    volumeMounts:
    - name: data-volume
      mountPath: /data
  volumes:
  - name: data-volume
    persistentVolumeClaim:
      claimName: large-data-pvc
```

## Testing

### Unit Tests
1. **PathMapper Component**: Test bidirectional path mapping with various container-host path scenarios, including edge cases like nested paths and symbolic links.
2. **Streaming Functions**: Verify `stream_file_as_pieces` and piece assembly logic with different file sizes and piece lengths.
3. **Command Parsing**: Test command-line argument validation for path mapping and streaming mode parameters.
4. **Error Handling**: Comprehensive testing of error scenarios including network failures, file access issues, and invalid configurations.

**Target Coverage**: >85% for all new components with focus on error paths and edge cases.

### Integration Tests
1. **End-to-End Streaming**: Verify complete upload/download workflows using `WritePersistentCacheTask` and `ReadPersistentCacheTask` interfaces with dfdaemon.
2. **Path Mapping Integration**: Test container-to-host path translation in realistic Kubernetes pod environments.
3. **Backward Compatibility**: Ensure existing dfcache functionality remains unaffected when new features are disabled.
4. **Performance Baseline**: Establish performance benchmarks comparing streaming vs non-streaming modes.

### Container Environment Tests
1. **Kubernetes Pod Testing**: Deploy dfcache in various Kubernetes configurations to validate path mapping accuracy.
2. **Volume Mount Scenarios**: Test different volume types (hostPath, PVC, configMap) with path mapping.
3. **Permission Validation**: Verify file access permissions work correctly across container-host boundaries.

### Performance and Stress Tests
1. **Large File Handling**: Test streaming performance with files ranging from 1GB to 100GB.
2. **Concurrent Operations**: Validate system stability under high-concurrency upload/download scenarios (50+ concurrent operations).
3. **Memory Usage**: Monitor memory consumption during large file transfers to prevent OOM conditions.
4. **Network Resilience**: Test streaming behavior under various network conditions including high latency and packet loss.

### Long-Duration Testing
- **7-day Continuous Operation**: Stress test with continuous file transfers to validate system stability.
- **Resource Leak Detection**: Monitor for memory leaks, file descriptor leaks, and goroutine leaks.

## Compatibility

### Backward Compatibility Guarantees
1. **Command Interface**: All existing dfcache import/export commands continue to work without modification.
2. **Configuration**: New features (streaming mode, path mapping) are **opt-in** via explicit command-line flags.
3. **Default Behavior**: Without new flags, dfcache behaves identically to current implementation.
4. **API Stability**: No changes to existing gRPC interfaces or data structures used by current functionality.

### Migration Strategy
1. **Gradual Adoption**: Users can enable streaming mode selectively for specific operations without affecting existing workflows.
2. **Configuration Validation**: Clear error messages for invalid path mapping formats or conflicting options.
3. **Fallback Mechanisms**: If streaming fails, system can fallback to legacy mode with appropriate logging.

### Version Compatibility
- **dfdaemon**: Requires dfdaemon version supporting `WritePersistentCacheTask` and `ReadPersistentCacheTask` interfaces.
- **Kubernetes**: Compatible with Kubernetes 1.20+ (no specific version requirements).
- **Container Runtimes**: Works with Docker, containerd, CRI-O without runtime-specific modifications.

### Configuration Management
```bash
# Legacy mode (default behavior)
dfcache import /path/to/file

# New streaming mode (opt-in)
dfcache import --stream-mode /path/to/file

# With path mapping (opt-in)
dfcache import --path-mapping="/data:/host/data" /data/file
```
