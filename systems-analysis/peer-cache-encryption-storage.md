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
    │   │   ├── piece_cryptor.rs
    │   └── algorithm/
    │       ├── mod.rs
    │       └── aes_ctr.rs
    └── ......
```

* `encrypt/cryptor/piece_cryptor.rs`: Add implementation of encryptor/decryptor.
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


### Configuration

Configuration options need to be added under `Config/Storage` to indicate whether encrypted storage is enabled and 
store the key from `Manager`.

```rust
// dragonfly-client-config/src/dfdaemon.rs
/// Encryption is the storage encryption configuration for dfdaemon.
#[derive(Debug, Clone, PartialEq, Eq, Serialize, Deserialize)]
pub struct Encryption {
    pub enable: bool,
    /// encryption_key is the global encryption key obtained from manager at runtime.  
    /// This field is not configurable via YAML and is populated dynamically.  
    #[serde(skip)]  
    pub key: Option<Vec<u8>>,
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


### Key Management

When the client starts, it sends an RPC request to the `Manager` to obtain a key, which will be used for cache encryption.
The client does not persist the key.

```rust
// dragonfly-client/src/bin/dfdaemon/main.rs
async fn main() -> Result<(), anyhow::Error> {
    ......
    // Initialize manager client.
    let manager_client = ManagerClient::new(...);
    let manager_client = Arc::new(manager_client);

    // Request a key from Manager
    let key = manager_client.request_encryption_key().await?;
    // Save key in config
    config.storage.encryption.set_key(key);
    ......
}
```

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


### Cryptor

The `PieceCryptor` trait defines the interface for the Cryptor:

```rust
// dragonfly-client-storage/src/encrypt/cryptor/piece_cryptor.rs

pub trait PieceCryptor {
    fn encrypt(&self, plaintext: &[u8], task_id: &str, piece_num: u32) -> Result<Vec<u8>>;
    fn decrypt(&self, ciphertext: &[u8], task_id: &str, piece_num: u32) -> Result<Vec<u8>>;
}
```

`task_id` and `piece_num` are used to construct the nonce.

The `encrypt` function returns an `Vec<u8>` as the encrypted `ciphertext`.

The `decrypt` function returns an `Vec<u8>` as the `plaintext`.

Below is an example implementation of `PieceCryptor` using the AES-CTR algorithm:

```rust
use aes::Aes256;
use ctr::Ctr128BE;

pub struct AesCtrCryptor {
    cipher: Ctr128BE<Aes256>,
}

impl AesCtrCryptor {
    pub fn new(key: &[u8]) -> Self {
        let key = Key::from_slice(key);
        Self {
            // use RustCrypto crate
            cipher: Ctr128BE<Aes256>::new(key),
        }
    }

    // construct nonce from task_id and piece_number, it does not need to be loaded/saved from/to disk
    fn build_nonce(task_id: &str, piece_num: u32) -> Vec<u8> {
        // Take certain bytes from task_id and piece_num to form the nonce.
        // For example, use the first 12 bytes of task_id and the first 4 bytes of piece_num to construct a 16-byte nonce.
        ......
    }

}

impl PieceCryptor for AesCtrCryptor {
    fn encrypt(&self, plaintext: &[u8], task_id: &str, piece_num: u32) -> Result<EncryptResult>{
        let nonce = Self::build_nonce(task_id, piece_num);
        
        // Construct the nonce and use the key to perform encryption.
        let res = self
            .cipher
            .encrypt(nonce, plaintext)?;
        
        Ok(res)
    }

    fn decrypt(&self, ciphertext: &[u8], task_id: &str, piece_num: u32) -> Result<Vec<u8>>{
        // The process of constructing the nonce is the same as in encryption.

        // decrypt
        let plaintext = self
            .cipher
            .decrypt(nonce, combined_slice)?;

        Ok(plaintext)
    }
}
```


### Piece Encryption/Decryption

The main changes are focused in `dragonfly-client-storage/src/lib.rs` and `dragonfly-client-storage/src/content.rs`.

When encryption is enabled, writing a `Piece` involves the following steps:
1. Read the plaintext of the `Piece` into memory
2. Calculate the CRC from the plaintext
3. Use the key obtained from the Manager and the constructed nonce to encrypt the data and obtain the ciphertext
4. Write the ciphertext to the same position in the file


```rust
// dragonfly-client-storage/src/content.rs
pub async fn write_persistent_cache_piece<R: AsyncRead + Unpin + ?Sized>(
    &self,
    ......
    // ADDED
    piece_id: &str,
) -> Result<WritePieceResponse> {
    // original code ↓
    // Open the file and seek to the offset.
    let task_path = self.get_persistent_cache_task_path(task_id);
    let mut f = OpenFile...?;

    f.seek(......)?;
    // original code ↑

    // ADDED need encrypt
    if self.config.storage.encryption.enable {
        // 1. read plaintext
        let mut plaintext = Vec::new();
        reader.read_to_end(&mut plaintext).await?;

        // 2. use plaintext to calculate crc
        let mut hasher = crc32fast::Hasher::new();
        hasher.update(&plaintext);
        let hash = hasher.finalize().to_string();

        // 3. encrypt
        let cryptor = get_cryptor(self.config.storage.encryption.key);
        let ciphertext = cryptor.encrypt_piece_by_id(&plaintext, piece_id)?;

        // 4. write
        let mut writer = BufWriter::with_capacity(self.config.storage.write_buffer_size, f);
        writer.write_all(&ciphertext).await?;
        writer.flush().await?;

        // check length
        if ciphertext.len() as u64 != expected_length {
            ......
        }

        Ok(WritePieceResponse {
            length: ciphertext.len() as u64,
            hash: hash,
        })

    } else {
        // if don not need encryption, do original process
        ......
        Ok(WritePieceResponse {
            length,
            hash: hasher.finalize().to_string(),
        })
    }
}
```

---

When encryption is enabled, reading a `Piece` involves the following steps:
1. Read the ciphertext of the `Piece` from the file into memory
2. Use the key obtained from the Manager and the constructed nonce to decrypt the data and obtain the plaintext

```rust
// dragonfly-client-storage/src/content.rs
pub async fn read_persistent_cache_piece(
    &self,
    ......
    // ADDED
    piece_id: &str,
) -> Result<impl AsyncRead> {
    // original code ↓
    let f = File::open(......)?;
    let mut f_reader = BufReader::with_capacity(self.config.storage.read_buffer_size, f);

    f_reader.seek(......)
    // original code ↑

    // ADDED
    let res = if self.config.storage.encryption.enable {
        // Read ciphertext from file
        let mut ciphertext = vec![0u8; target_length as usize];
        f_reader.read_exact(&mut ciphertext).await?;

        // decryption
        let cryptor = get_cryptor(self.config.storage.encryption.key);
        let plaintext = cryptor.decrypt_piece_by_id(&ciphertext, piece_id)?;

        Either::Left(Cursor::new(plaintext))
    } else {
        Either::Right(f_reader.take(target_length))
    };

    Ok(res)
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