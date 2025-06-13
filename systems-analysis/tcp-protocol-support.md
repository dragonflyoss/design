# TCP Protocol Support for P2P File Transfer

## Overview

This design document proposes adding TCP protocol support to Dragonfly's P2P file transfer mechanism. Currently, Dragonfly only supports gRPC protocol for peer-to-peer communication. Adding TCP support will provide a more lightweight transport option that may offer better performance in certain network environments.

## Motivation

- **Performance**: TCP direct connection reduces protocol stack overhead compared to gRPC
- **Flexibility**: Multiple transport protocols increase network adaptability
- **Compatibility**: TCP support enables better compatibility with diverse network environments
- **Optimization**: Opportunity for protocol-specific optimizations for large files/AI models

## Design Goals

1. Add TCP protocol support while maintaining backward compatibility with existing gRPC implementation
2. Preserve the existing `Downloader` interface for seamless integration
3. Support Dragonfly's Vortex application-layer protocol over TCP
4. Optimize for large file transfers and maximize network bandwidth utilization

## Architecture

### Current Architecture
```
DownloaderFactory -> Downloader (Interface) -> GRPCDownloader
```

### Proposed Architecture
```
DownloaderFactory -> Downloader (Interface) -> {
    GRPCDownloader,
    TCPDownloader
}
```

### Module Structure

#### New Modules
```
dragonfly-client-storage/src/
├── client/
│   ├── mod.rs          # Client trait definition
│   └── tcp.rs          # TCPClient implementation
└── server/
    ├── mod.rs          # Server trait definition
    └── tcp.rs          # TCPServer implementation
```

#### Modified Modules
- `piece_downloader.rs`: Add TCP support to DownloaderFactory
- `dfdaemon/main.rs`: Add protocol configuration parsing
- Configuration file: Add `storage.server.protocol` option

## Technical Implementation

### 1. Core Interfaces

#### Client Trait
```rust
#[async_trait]
pub trait Client: Send + Sync {
    type Config;
    type Error;
    
    async fn new(
        config: Self::Config,
        addr: String,
        is_download_piece: bool
    ) -> Result<Self, Self::Error> where Self: Sized;
    
    async fn download_piece(
        &self,
        request: DownloadPieceRequest,
        timeout: Duration,
    ) -> Result<DownloadPieceResponse, Self::Error>;
    
    async fn download_persistent_cache_piece(
        &self,
        request: DownloadPersistentCachePieceRequest,
        timeout: Duration,
    ) -> Result<DownloadPersistentCachePieceResponse, Self::Error>;
}
```

### 2. Vortex Protocol Adaptation

The TCP implementation will support Dragonfly's Vortex application-layer protocol:

#### Message Format
```
+--------+--------+--------+--------+
| ID     | Tag    | Length | Value  |
| (1B)   | (1B)   | (4B)   | (var)  |
+--------+--------+--------+--------+
```

#### Message Types
- **Download Piece (Tag=0x00)**: Request piece content from peer
- **Piece Content (Tag=0x01)**: Raw piece data or fragment
- **Error (Tag=0xFF)**: Data transmission error
- **Close (Tag=0xFE)**: Connection termination
- **Reserved Tags (2-253)**: Future protocol extensions

#### Encoding/Decoding Implementation
```rust
impl Vortex {
    pub fn from_bytes(bytes: Bytes) -> Result<Self> {
        // Parse header and validate packet structure
        // Extract packet_id, tag, length, and value
        // Return appropriate Vortex enum variant
    }

    pub fn to_bytes(&self) -> Bytes {
        // Serialize packet to byte format
        // Construct header with packet_id, tag, length
        // Append value bytes
    }
}
```

### 3. TCP Downloader Implementation

```rust
pub struct TCPDownloader {
    config: Arc<Config>,
    clients: Arc<Mutex<HashMap<String, TCPClientEntry>>>,
    capacity: usize,
    idle_timeout: Duration,
    cleanup_at: Arc<Mutex<Instant>>,
}

struct TCPClientEntry {
    client: TCPClient,
    active_requests: Arc<AtomicUsize>,
    actived_at: Arc<std::sync::Mutex<Instant>>,
}

#[async_trait]
impl Downloader for TCPDownloader {
    async fn download_piece(
        &self,
        addr: &str,
        number: u32,
        host_id: &str,
        task_id: &str,
    ) -> Result<(Cursor<Vec<u8>>, u64, String)> {
        // 1. Get or create TCP client for addr
        // 2. Send download_piece request via Vortex protocol
        // 3. Receive and validate response
        // 4. Return piece data with metadata
    }
}
```

### 4. Performance Optimizations

#### Zero-Copy Implementation
Using `sendfile` for efficient data transfer:

```rust
/// read_piece reads the piece from the content.
    #[instrument(skip_all)]
    pub async fn read_piece(
        &self,
        socket_fd: c_int,
        task_id: &str,
        offset: u64,
        length: u64,
        range: Option<Range>,
    ) {
        let task_path = self.get_task_path(task_id);

        // Calculate the target offset and length based on the range.
        let (target_offset, target_length) = calculate_piece_range(offset, length, range);

        let f = File::open(task_path.as_path()).await.inspect_err(|err| {
            error!("open {:?} failed: {}", task_path, err);
        })?;
        
        // Original Implementation
        // let mut f_reader = BufReader::with_capacity(self.config.storage.read_buffer_size, f); 
        // f_reader
        //     .seek(SeekFrom::Start(target_offset))
        //     .await
        //     .inspect_err(|err| {
        //         error!("seek {:?} failed: {}", task_path, err);
        //     })?;
        // Ok(f_reader.take(target_length))
        
        //Sendfile Implementation
        task::spawn_blocking(move || {
            unsafe {
                libc::sendfile(socket_fd, f, target_offset, target_length)
            }
        }).await?;
    }
```

#### TCP Tuning Parameters
- **Buffer Sizes**: Optimize SO_RCVBUF and SO_SNDBUF
- **TCP Options**: TCP_NODELAY, TCP_CORK, TCP_KEEPALIVE
- **Congestion Control**: BBR algorithm for high-bandwidth networks
- **Connection Management**: Connection pooling with idle timeout

### 5. Configuration

Add TCP protocol configuration to `dfdaemon.yaml`:

```yaml
storage:
  server:
    tcp:
      port: 4005
      buffer_size: 4194304
      idle_timeout: 300s
      max_connections: 1000
```

## Testing Strategy

1. **Unit Tests**: Individual component testing with 85%+ coverage
2. **Integration Tests**: End-to-end functionality verification
3. **Performance Tests**: TCP vs gRPC benchmarking
4. **Stress Tests**: High concurrency and long-duration testing

## Expected Outcomes

1. **Functionality**: Complete TCP-based P2P file transfer capability
2. **Performance**: 10-20% performance improvement in specific scenarios
4. **Documentation**: Comprehensive user and developer documentation

## Compatibility

- **Backward Compatibility**: Existing gRPC functionality remains unchanged
- **Configuration**: New TCP options with sensible defaults
- **Migration**: Users can switch protocols via configuration without code changes

## Future Considerations

- **Protocol Negotiation**: Automatic protocol selection based on network conditions
- **Hybrid Mode**: Simultaneous multi-protocol support for optimal performance

---

This design provides a foundation for adding TCP protocol support to Dragonfly while maintaining system stability and backward compatibility.
