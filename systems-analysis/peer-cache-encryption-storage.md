# Peer Cache Encryption Storage

## Overview

This design document proposes adding encryption storage for peer cache in Dragonfly's P2P file transfer mechanism. The goal is to enhance data security for cached files within the P2P network, ensuring that sensitive information remains protected during storage and transfer.

## Motivation

  * **Data Security**: Encrypting cached data protects sensitive information from unauthorized access, especially in environments where peers might store data on insecure disks.
  * **Compliance**: Addresses potential compliance requirements for data at rest and in transit within the P2P network.

## Goals

1.  Introduce encryption and decryption capabilities for peer-cached data.
2.  Maintain backward compatibility with existing Dragonfly functionalities.
3.  Support configurable encryption options, allowing users to enable/disable and select encryption algorithms.
4.  Integrate seamlessly with the existing Persistent Cache workflow.
5.  Ensure minimal performance overhead for encryption/decryption operations.

## Architecture

The proposed architecture involves introducing an encryption module within the Dragonfly client's storage layer. This module will intercept data during writing to and reading from the persistent cache, performing encryption and decryption operations respectively.

```
P2P Network -> Downloader -> Piece Writer (Encryption) -> Persistent Cache Storage
Persistent Cache Storage -> Piece Reader (Decryption) -> Downloader -> P2P Network
```

### Modules

A new module `encrypt` will be added in `dragonfly-client-storage` to handle encryption-related functionality.

```
dragonfly-client-storage/
└── src/
    ├── encrypt/
    │   ├── mod.rs
    │   ├── cryptor/
    │   │   ├── mod.rs
    │   │   ├── reader.rs
    │   └── algorithm/
    │       ├── mod.rs
    │       └── aes_ctr.rs
    └── ......
```

* `encrypt/cryptor/reader.rs`: Add implementation of encryptor/decryptor.
* `encrypt/algorithm/*.rs`: Add implementation of specific encryption algorithms.

## Implementation

### Encryption Algorithm

Cache may involve large files, in which case symmetric encryption is usually more efficient and simpler to manage.

Currently, commonly used algorithms are AES and ChaCha20, each with some enhanced versions:


| Property/Algorithm | AES-CTR | AES-GCM | ChaCha20 | XChaCha20 | ChaCha20-Poly1305 |
| :----------------- | :------ | :------ | :------- | :-------- | :---------------- |
| Algorithm Type     | Stream Cipher (non-authenticated) | AEAD (Authenticated) | Stream Cipher (non-authenticated) | Stream Cipher (non-authenticated) | AEAD (Authenticated) |
| Authentication (Integrity Check) | None | Yes | None | None | Yes |
| Content to be saved for decryption | key + iv | key + iv + tag (per block) | key + nonce | key + nonce | key + nonce + tag (per block) |
| Nonce/IV Length | 16 bytes (IV) | 12 bytes | 12 bytes | 24 bytes | 12 bytes |
| Speed (with hardware support) | Very Fast | Fast (has authentication cost) | Fast | Fast | Moderate |
| Speed (without hardware support) | Slow | Slow | Still fast | Still fast | Still Moderate |
| Hardware Acceleration Support | Widely supports AES-NI | Widely supports AES-NI | Relies mainly on SSE/AVX | Relies mainly on SSE/AVX | Relies mainly on SSE/AVX |
| Security | No authentication (easily tampered) | High | No authentication | No authentication | High |
| Common Uses | Disk cache, fast encryption | TLS, QUIC, HTTPS | WireGuard, embedded | WireGuard, large file streaming | age, Cloudflare file encryption |
| Ciphertext length equal to plaintext | Yes | No (+tag) | Yes | Yes | No (+tag) |


The advantage of AES is its more widespread hardware acceleration support, making it very fast when hardware support is available. 

However, on platforms without hardware support, AES's performance might not be as good as ChaCha20.

In addition to encryption, authentication can detect whether data has been tampered with, but it requires storing additional authentication information. AES-GCM and ChaCha20-Poly1305 have this capability.


Dragonfly's current design writes Pieces to its corresponding offsets in the final file. This implies that each Piece must be encrypted prior to being written to disk to ensure no plaintext information is stored during the entire process.


In this case, authenticated algorithms would introduce additional complexity because authentication information would need to be saved for each encrypted Piece block.

Given that Dragonfly is typically deployed on servers, it is highly likely to have AES hardware acceleration, so AES speed should be optimal. 

At the same time, Dragonfly uses CRC32 and other verification methods, which to some extent defend against tampering (but not completely). 

In this case, authentication may not be the primary concern, so AES-CTR is a good choice.


### Manager: Key Management


The manager is responsible for storing keys, and clients obtain keys from the manager through RPC requests.

Users can specify a key in the manager's config using base64 encoding. The manager will save this key to the database and overwrite any existing key in the database. (The key in the config file has higher priority than the key in the database.)

If the user does not specify a key, the manager will use the key from the database. If there is no key in the manager's database, it will randomly generate one, use it, and save it.

The manager will perform this work after initializing the database.

The new RPC will be defined in `dragonfly-api`.

```
// dragonfly-api/pkg/apis/manager/v2/manager.proto

// RequestEncryptionKeyRequest represents request of RequestEncryptionKey.  
message RequestEncryptionKeyRequest {  
  // Request source type.  
  SourceType source_type = 1 [(validate.rules).enum.defined_only = true];  
  // Source service hostname.  
  string hostname = 2 [(validate.rules).string.hostname = true];  
  // Source service ip.  
  string ip = 3 [(validate.rules).string.ip = true];  
}  
  
// RequestEncryptionKeyResponse represents response of RequestEncryptionKey.  
message RequestEncryptionKeyResponse {  
  // Encryption key provided by manager.  
  bytes encryption_key = 1;  
}
```

### Client: Configuration

Configuration options need to be added under `Config/Storage` to indicate whether encrypted storage is enabled.

```rust
// dragonfly-client-config/src/dfdaemon.rs
/// Encryption is the storage encryption configuration for dfdaemon.
#[derive(Debug, Clone, PartialEq, Eq, Serialize, Deserialize)]
pub struct Encryption {
    pub enable: bool,
}

pub struct Storage {
    ......
    #[serde(default = "default_storage_encryption")] 
    pub encryption: Encryption,
}
```

Config yaml example:

```yaml
storage:
  dir: /var/lib/dragonfly/
  keep: true
  writeBufferSize: 4194304
  readBufferSize: 4194304
  # ADDED
  encryption:
    enable: true
```


### Client: Request Key

When the client starts, it sends an RPC request to the `Manager` to obtain a key, which will be used for cache encryption.

The client does not persist the obtained key, so it will request a key from the manager every time it restarts.

```rust
// dragonfly-client/src/bin/dfdaemon/main.rs
async fn main() -> Result<(), anyhow::Error> {
    ......
    // Initialize manager client.
    let manager_client = ManagerClient::new(...);
    let manager_client = Arc::new(manager_client);

    // Request a key from Manager
    let key = manager_client.request_encryption_key().await?;
    // Pass key as a parameter of Storage
    let storage = Storage::new(..., key).await?;
    ......
}
```

### Client: Cryptor

The `EncryptAlgo` trait defines the interface for the Encryption alogorithm.

```rust
// dragonfly-client-storage/src/encrypt/algorithm/mod.rs

pub trait EncryptAlgo {
    const NONCE_SIZE: usize;
    const KEY_SIZE: usize;

    fn new(key: &[u8], nonce: &[u8]) -> Self;

    fn apply_keystream(&mut self, data: &mut [u8]);

    fn build_nonce(task_id: &str, piece_num: u32);
}
```

`task_id` and `piece_num` are used to construct the nonce.

The `apply_keystream` function performs in-place encryption/decryption on `data`.

---

`EncryptReader`/`DecryptReader` provides asynchronous encryption/decryption capabilities. 
It can adapt well to existing impl AsyncRead parameter types.

```rust
// dragonfly-client-storage/src/encrypt/cryptor/reader.rs
pub struct EncryptReader<R, A: EncryptAlgo> {
    inner: R,
    cipher: A,
}

impl<R: AsyncRead + Unpin, A: EncryptAlgo> AsyncRead for EncryptReader<R, A> {
    // implement AsyncRead
    ......
}
```



### Client: Piece Encryption/Decryption

The main changes are focused in `dragonfly-client-storage/src/lib.rs` and `dragonfly-client-storage/src/content.rs`.

When encryption is enabled, writing a `Piece` involves the following steps:
1. Get the key for encryption
2. Create an `EncryptReader` to wrap the original `impl AsyncRead` parameter
3. Use the `EncryptReader` to copy the ciphertext to the corresponding position in the file

```rust
// dragonfly-client-storage/src/content.rs
pub async fn write_persistent_cache_piece<R: AsyncRead + Unpin + ?Sized>(
    &self,
    ......
    // ADDED
    piece_id: &str,
) -> Result<WritePieceResponse> {
    // original code ↓
    // Open the target file and set CRC32 hasher
    let writer = ...
    let tee = hasher_inspect_reader();
    // original code ↑

    // Modified
    let mut tee_warpper = if self.config.storage.encryption.enable {
        // 1. get key
        let key = get_key();
        // 2. use EncryptReader
        let encrypt_reader = EncryptReader::new(tee, key, ...);
        Either::Left(encrypt_reader)
    } else {
        Either::Right(tee)
    };

    // 3. copy to file
    let length = io::copy(&mut tee_warpper, &mut writer).await.inspect_err(|err| {
        error!("copy {:?} failed: {}", task_path, err);
    })?;
    
    ......
}
```

---

When encryption is enabled, reading a `Piece` involves the following steps:
1. Get the key for encryption
2. Create an `DecryptReader` to wrap the original `impl AsyncRead` parameter
3. Return the `DecryptReader`

```rust
// dragonfly-client-storage/src/content.rs
pub async fn read_persistent_cache_piece(
    &self,
    ......
    // ADDED
    piece_id: &str,
) -> Result<impl AsyncRead> {
    // original code ↓
    // Open file
    let f_reader = ...;
    // original code ↑

    // ADDED
    if self.config.storage.encryption.enable {
        // 1. get key
        let key = get_key();
        
        let limited_reader = f_reader.take(target_length);
        // 2. use DecryptReader
        let decrypt_reader = DecryptReader::new(
            limited_reader, 
            key, 
            piece_id
        );

        // 3. return DecryptReader
        return Ok(Either::Left(decrypt_reader));
    }

    Ok(Either::Right(f_reader.take(target_length)))
}
```


## Testing

1.  **Unit Tests**: Test individual components of the encryption module with high coverage (over 85%).
2.  **Integration Tests**: End-to-end functional verification, ensuring that data is correctly encrypted and decrypted during the P2P transfer process.
3.  **Performance Tests**: Benchmark the performance impact of encryption/decryption on file transfer speeds compared to unencrypted transfers.
4.  **Stress Tests**: High-concurrency and long-duration testing to ensure stability and reliability with encryption enabled.

## Compatibility

1.  **Backward Compatibility**: The existing `Task` and `PersistentCacheTask` functionalities will remain unchanged for non-encrypted operations. The encryption feature will be optional and configurable.
2.  **Configuration**: New encryption-related configurations will be added with sensible defaults (e.g., encryption disabled by default).
3.  **Migration**: Users can enable or disable the encryption feature via configuration without requiring code changes.

## Future

1.  **Key Management Enhancements**: Explore integration with Key Management Systems (KMS) for more robust key handling.
2.  **Algorithm Flexibility**: Support for a wider range of encryption algorithms and modes.
3.  **Dynamic Key Rotation**: Implement mechanisms for automatic key rotation to enhance security.

-----

This design provides a foundation for adding P2P Peer Cache Encryption Storage to Dragonfly while maintaining system stability and backward compatibility.

<!-- **Issue Link**: [Encrypted storage for P2P peer node cache.](https://github.com/dragonflyoss/dragonfly/issues/4026) -->


# TODO

- [ ] Graph in Architecture
- [ ] Normal `Task` Implementation

