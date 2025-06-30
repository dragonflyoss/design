# QUIC Protocol Support for P2P File Transfer

## Overview

This design document proposes adding QUIC protocol support to Dragonfly's P2P file transfer mechanism. Currently, Dragonfly only supports gRPC protocol for peer-to-peer communication. Adding QUIC support will provide a more lightweight transport option that may offer better performance in certain network environments.

## Motivation

- **Performance**: 0-RTT connection reduces stack overhead compared to gRPC
- **Flexibility**: Multiple transport protocols increase network adaptability
- **Compatibility**: QUIC support enables better compatibility with diverse network environments
- **Optimization**: Opportunity for protocol-specific optimizations for large files/AI models

## Goals

1. Add QUIC protocol support while maintaining backward compatibility with existing gRPC implementation
2. Preserve the existing `Downloader` interface for seamless integration
3. Support Dragonfly's Vortex application-layer protocol over QUIC
4. Optimize for large file transfers and maximize network bandwidth utilization

## Architecture

```
DownloaderFactory -> Downloader (Interface) -> {
    GRPCDownloader,
    QUICDownloader
}
```

### Modules

```
dragonfly-client-storage/src/
├── client/
│   ├── mod.rs          # Client trait definition
│   └── quic.rs         # QUICClient implementation
└── server/
    ├── mod.rs          # Server trait definition
    └── quic.rs         # QUICServer implementation
```

- `piece_downloader.rs`: Add QUIC support to DownloaderFactory
- `dfdaemon/main.rs`: Add protocol configuration parsing
- Configuration file: Add `storage.server.protocol` option
  
## Implementation

### Client

```rust
#[async_trait]
/// StorageClient is the client for downloading pieces from other peers.
pub struct StorageClient {
    /// config is the configuration of the dfdaemon.
    config: Arc<Config>,
}

impl StorageClient {
    /// new creates a new storage client.
    pub fn new(config: Arc<Config>) -> Self {
        Self { config }
    }

    /// download_piece downloads a piece from a peer.
    #[instrument(skip_all)]
    pub async fn download_piece(
        &self,
        peer_id: &str,
        task_id: &str,
        piece_id: &str,
        range: Option<Range>,
    ) -> Result<impl AsyncRead> {
        match self.config.storage.server.protocol.as_str() {
            "grpc" => self.download_piece_via_grpc(peer_id, task_id, piece_id, range).await,
            "quic" => self.download_piece_via_quic(peer_id, task_id, piece_id, range).await,
            _ => Err(Error::InvalidParameter),
        }
    }

    /// download_piece_via_quic downloads a piece via QUIC.
    #[instrument(skip_all)]
    async fn download_piece_via_quic(
        &self,
        peer_id: &str,
        task_id: &str,
        piece_id: &str,
        range: Option<Range>,
    ) -> Result<impl AsyncRead> {
        // TODO: Implement QUIC client for downloading pieces
        Err(Error::NotImplemented)
    }
}
```

### Vortex Protocol 

The QUIC implementation will support Dragonfly's Vortex application-layer protocol:

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

### QUIC Downloader 

```rust
pub struct QUICDownloader {
    config: Arc<Config>,
    clients: Arc<Mutex<HashMap<String, QUICClientEntry>>>,
    capacity: usize,
    idle_timeout: Duration,
    cleanup_at: Arc<Mutex<Instant>>,
}

struct QUICClientEntry {
    client: QUICClient,
    active_requests: Arc<AtomicUsize>,
    actived_at: Arc<std::sync::Mutex<Instant>>,
}

#[async_trait]
impl Downloader for QUICDownloader {
    async fn download_piece(
        &self,
        addr: &str,
        number: u32,
        host_id: &str,
        task_id: &str,
    ) -> Result<(Cursor<Vec<u8>>, u64, String)> {
        // 1. Get or create QUIC client for addr
        // 2. Send download_piece request via Vortex protocol
        // 3. Receive and validate response
        // 4. Return piece data with metadata
    }
}
```

### Performance 

#### Zero-Copy

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

#### Quinn Configuration

Using `quinn` for efficient data transfer:

##### Server

```rust
let mut server_config = quinn::ServerConfig::with_single_cert(cert_chain, key)?;
server_config.transport = optimized_quinn_transport_config();
let endpoint = quinn::Endpoint::server(server_config, listen_addr)?;
```
##### Client

```rust
let mut client_config = quinn::ClientConfig::default();
client_config.transport_config(optimized_quinn_transport_config());
let mut endpoint = quinn::Endpoint::client("[::]:0".parse().unwrap())?;
endpoint.set_default_client_config(client_config);
```

#### Quinn Parameters

- **Flow Control**: Each connection supports multiple concurrent streams, dynamically adjust flow control window size and multiplex multiple data streams on a single connection
- **Connection Management**: Avoid the overhead of frequently establishing and destroying connections
 , automatically clean up idle connections,prevent resource exhaustion from too many connections
- **Congestion Control**: BBR algorithm and cubic for high-bandwidth networks 
- **0-RTT**: Enable 0-RTT to reduce connection establishment time

### Configuration

Add QUIC protocol configuration to `dfdaemon.yaml`:

```yaml
storage:
  server:
    quic:
      port: 4006
      buffer_size: 4194304
      idle_timeout: 300s
      max_connections: 1000
```

## Testing 

1. **Unit Tests**: Individual component testing with 85%+ coverage
2. **Integration Tests**: End-to-end functionality verification
3. **Performance Tests**: QUIC vs gRPC benchmarking
4. **Stress Tests**: High concurrency and long-duration testing

## Compatibility

- **Backward Compatibility**: Existing gRPC functionality remains unchanged
- **Configuration**: New QUIC options with sensible defaults
- **Migration**: Users can switch protocols via configuration without code changes

## Future

- **Protocol Negotiation**: Automatic protocol selection based on network conditions
- **Hybrid Mode**: Simultaneous multi-protocol support for optimal performance

---

This design provides a foundation for adding QUIC protocol support to Dragonfly while maintaining system stability and backward compatibility.
