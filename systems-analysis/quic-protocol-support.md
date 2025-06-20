# QUIC Protocol Support for P2P File Transfer

## Overview

This design document proposes adding QUIC protocol support to Dragonfly's P2P file transfer mechanism. Currently, Dragonfly only supports gRPC protocol for peer-to-peer communication. Adding QUIC support will provide a more lightweight transport option that may offer better performance in certain network environments.

## Motivation

- **Performance**: QUIC integrates TLS 1.3 encryption and supports 0-RTT connection establishment, reducing the handshake overhead compared to gRPC
- **Flexibility**: Multiple transport protocols increase network adaptability
- **Compatibility**: QUIC support enables better compatibility with diverse network environments
- **Optimization**: QUIC can better handle packet loss, high latency, and network jitter by dynamically adjusting transmission parameters based on real-time network metrics. 

## Design Goals

1. Add QUIC protocol support while maintaining backward compatibility with existing gRPC implementation
2. Preserve the existing `Downloader` interface for seamless integration
3. Support Dragonfly's Vortex application-layer protocol over QUIC
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
    QUICDownloader
}
```

### Module Structure

#### New Modules
```
dragonfly-client-storage/src/
├── client/
│   ├── mod.rs          # Client trait definition
│   └── quic.rs         # quicClient implementation
└── server/
    ├── mod.rs          # Server trait definition
    └── quic.rs         # quicServer implementation
```

#### Modified Modules
- `piece_downloader.rs`: Add QUIC support to DownloaderFactory
- `dfdaemon/main.rs`: Add protocol configuration parsing
- Configuration file: Add `storage.server.protocol` option

## Technical Implementation

### 1. Protocol Support



Adding documentation for the QUIC protocol and default port configuration for the QUIC server:
```rust
pub struct StorageServer {
    pub protocol: String,

    /// quic_port is the port for QUIC server when protocol is set to "quic".
    #[serde(default = "default_storage_quic_port")]
    pub quic_port: u16,
}

/// default_storage_quic_port is the default port for QUIC server.
#[inline]
fn default_storage_quic_port() -> u16 {
    4005
}

/// Storage implements Default.
impl Default for StorageServer {
    fn default() -> Self {
        StorageServer {
            protocol: default_storage_server_protocol(),
            quic_port: default_storage_quic_port(),
        }
    }
}
```

### 2. Vortex Protocol Adaptation

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

### 3. QUIC Downloader Implementation

```rust
/// StorageServer is the server for uploading pieces to other peers.
pub struct StorageServer {
    /// config is the configuration of the dfdaemon.
    config: Arc<Config>,
}

impl StorageServer {
    /// new creates a new storage server.
    pub fn new(config: Arc<Config>) -> Self {
        Self { config }
    }

    /// start starts the storage server.
    #[instrument(skip_all)]
    pub async fn start(&self) -> Result<()> {
        match self.config.storage.server.protocol.as_str() {
            "grpc" => self.start_grpc_server().await,
            "quic" => self.start_quic_server().await,
            _ => Err(Error::InvalidParameter),
        }
    }

    /// start_grpc_server starts the gRPC server.
    #[instrument(skip_all)]
    async fn start_grpc_server(&self) -> Result<()> {
        // TODO: Implement gRPC server for uploading pieces
        Err(Error::NotImplemented)
    }

    /// start_quic_server starts the QUIC server.
    #[instrument(skip_all)]
    async fn start_quic_server(&self) -> Result<()> {
        // TODO: Implement QUIC server for uploading pieces
        Err(Error::NotImplemented)
    }

    /// upload_piece uploads a piece to a peer.
    #[instrument(skip_all)]
    pub async fn upload_piece(
        &self,
        task_id: &str,
        piece_id: &str,
        range: Option<Range>,
    ) -> Result<impl AsyncRead> {
        match self.config.storage.server.protocol.as_str() {
            "grpc" => self.upload_piece_via_grpc(task_id, piece_id, range).await,
            "quic" => self.upload_piece_via_quic(task_id, piece_id, range).await,
            _ => Err(Error::InvalidParameter),
        }
    }

    /// upload_piece_via_grpc uploads a piece via gRPC.
    #[instrument(skip_all)]
    async fn upload_piece_via_grpc(
        &self,
        task_id: &str,
        piece_id: &str,
        range: Option<Range>,
    ) -> Result<impl AsyncRead> {
        // TODO: Implement gRPC server for uploading pieces
        Err(Error::NotImplemented)
    }

    /// upload_piece_via_quic uploads a piece via QUIC.
    #[instrument(skip_all)]
    async fn upload_piece_via_quic(
        &self,
        task_id: &str,
        piece_id: &str,
        range: Option<Range>,
    ) -> Result<impl AsyncRead> {
        // TODO: Implement QUIC server for uploading pieces
        Err(Error::NotImplemented)
    }
}
```

### 4. Performance Optimizations

#### Congestion Control Tuning


```rust
async fn tune_congestion_control(&self, metrics: &QuicMetrics) -> Result<()> {
    let mut base_config = &mut self.tuning_config.base_config;

    // Adjust the congestion window based on the packet loss rate
    if metrics.packet_loss_rate > 0.1 {
        // High packet loss rate, reduce the congestion window
        base_config.max_congestion_window = (base_config.max_congestion_window as f64 * 0.8) as u32;
        base_config.initial_congestion_window = (base_config.initial_congestion_window as f64 * 0.8) as u32;
        warn!("Reducing congestion window due to high packet loss rate: {}", metrics.packet_loss_rate);
    } else if metrics.packet_loss_rate < 0.01 && metrics.throughput < base_config.max_congestion_window as u64 {
        // Low packet loss rate and throughput not reaching the upper limit, increase the congestion window
        base_config.max_congestion_window = (base_config.max_congestion_window as f64 * 1.2) as u32;
        base_config.initial_congestion_window = (base_config.initial_congestion_window as f64 * 1.2) as u32;
        debug!("Increasing congestion window due to low packet loss rate: {}", metrics.packet_loss_rate);
    }

    Ok(())
}
```
#### Flow Control Tuning


```rust
async fn tune_flow_control(&self, metrics: &QuicMetrics) -> Result<()> {
    let mut base_config = &mut self.tuning_config.base_config;

    // Adjust the maximum number of concurrent streams based on the number of active streams
    if metrics.active_streams > base_config.max_concurrent_streams as u64 * 80 / 100 {
        // The number of active streams is close to the upper limit, increase the maximum number of concurrent streams
        base_config.max_concurrent_streams = (base_config.max_concurrent_streams as f64 * 1.2) as u32;
        debug!("Increasing max concurrent streams due to high active streams: {}", metrics.active_streams);
    } else if metrics.active_streams < base_config.max_concurrent_streams as u64 * 20 / 100 {
        // The number of active streams is far below the upper limit, reduce the maximum number of concurrent streams
        base_config.max_concurrent_streams = (base_config.max_concurrent_streams as f64 * 0.8) as u32;
        debug!("Decreasing max concurrent streams due to low active streams: {}", metrics.active_streams);
    }

    Ok(())
}
```
#### Retransmission Tuning


```rust
async fn tune_retransmission(&self, metrics: &QuicMetrics) -> Result<()> {
    let mut base_config = &mut self.tuning_config.base_config;

    // Adjust RTT and ACK delay based on the retransmission rate
    if metrics.retransmission_rate > 0.1 {
        // High retransmission rate, increase RTT estimate and ACK delay
        base_config.initial_rtt = (base_config.initial_rtt as f64 * 1.2) as u32;
        base_config.max_ack_delay = (base_config.max_ack_delay as f64 * 1.2) as u32;
        warn!("Increasing RTT and ACK delay due to high retransmission rate: {}", metrics.retransmission_rate);
    } else if metrics.retransmission_rate < 0.01 {
        // Low retransmission rate, reduce RTT estimate and ACK delay
        base_config.initial_rtt = (base_config.initial_rtt as f64 * 0.8) as u32;
        base_config.max_ack_delay = (base_config.max_ack_delay as f64 * 0.8) as u32;
        debug!("Decreasing RTT and ACK delay due to low retransmission rate: {}", metrics.retransmission_rate);
    }

    Ok(())
}
```

#### QUIC Optimization Parameters
- **Concurrency Optimization**: Enable 0-RTT connections to reduce handshake latency
- **Transmission Optimization**: Ajust the maximum ACK delay to reduce acknowledgment latency
- **Congestion Control**: Adjust the congestion window to ensure basic transmission efficiency and make full use of bandwidth
- **Connection Management**: Connection pooling with idle timeout

### 5. Configuration

Add QUIC protocol configuration to `dfdaemon.yaml`:

```yaml
storage:
  server:
    QUIC:
      port: 4005
      buffer_size: 4194304
      idle_timeout: 300s
      max_connections: 1000
```
Set the tuning parameters:


```rust
impl Default for QuicConfig {
    fn default() -> Self {
        Self {
            max_concurrent_streams: 100,
            initial_congestion_window: 10 * 1024, // 10KB
            max_congestion_window: 10 * 1024 * 1024, // 10MB
            min_congestion_window: 2 * 1024, // 2KB
            initial_rtt: 100,
            max_ack_delay: 25,
            idle_timeout: 30,
            max_udp_payload_size: 1350,
            enable_0rtt: true,
        }
    }
}
```
## Testing Strategy

1. **Unit Tests**: Individual component testing with 85%+ coverage
2. **Integration Tests**: End-to-end functionality verification
3. **Performance Tests**: QUIC vs gRPC benchmarking
4. **Stress Tests**: High concurrency and long-duration testing

## Expected Outcomes

1. **Functionality**: Complete QUIC-based P2P file transfer capability
2. **Performance**: 10-20% performance improvement in specific scenarios
4. **Documentation**: Comprehensive user and developer documentation

## Compatibility

- **Backward Compatibility**: Existing gRPC functionality remains unchanged
- **Configuration**: New QUIC options with sensible defaults
- **Migration**: Users can switch protocols via configuration without code changes

## Future Considerations

- **Protocol Negotiation**: Automatic protocol selection based on network conditions
- **Hybrid Mode**: Simultaneous multi-protocol support for optimal performance

---

This design provides a foundation for adding QUIC protocol support to Dragonfly while maintaining system stability and backward compatibility.
