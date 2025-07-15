# Design Document: P2P Peer Cache Encryption Storage

## Overview

This design document proposes adding encryption storage for peer cache in Dragonfly's P2P file transfer mechanism. The goal is to enhance data security for cached files within the P2P network, ensuring that sensitive information remains protected during storage and transfer.

## Motivation

  * **Data Security**: Encrypting cached data protects sensitive information from unauthorized access, especially in environments where peers might store data on insecure disks.
  * **Compliance**: Addresses potential compliance requirements for data at rest and in transit within the P2P network.
  * **Integrity**: Helps ensure the integrity of cached data by detecting tampering attempts.

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

A new module `dragonfly-client-crypto` will be added to handle encryption-related functionality.

```
dragonfly-client-crypto/
└── src/
    ├── cryptor/
    │   ├── crypto_type.rs
    │   ├── piece_cryptor.rs
    │   └── mod.rs
    ├── algorithm/
    │   ├── chacha20poly1305.rs
    │   ├── ...other_algorithms.rs
    │   └── mod.rs
    └── lib.rs
```

* `cryptor/crypto_type.rs`: Defines the types of encryption.
* `cryptor/piece_cryptor.rs`: Add implementation of encryptor/decryptor.
* `algorithm/*.rs`: Add implementation of specific encryption algorithms.


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

Besides encryption, authentication can check if data has been maliciously tampered with, but at the cost of needing to store additional authentication information. AES-GCM and ChaCha20-Poly1305 have this capability.


Dragonfly's current design writes Pieces to their corresponding offsets in the final file. This implies that each Piece must be encrypted prior to being written to disk to ensure no plaintext information is stored during the entire process.


In this case, authenticated algorithms would introduce additional complexity because authentication information would need to be saved for each encrypted Piece block.

Given that Dragonfly is typically deployed on servers, it is highly likely to have AES hardware acceleration, so AES speed should be optimal. 

At the same time, Dragonfly uses CRC32 and other verification methods, which to some extent defend against tampering (but not completely). Authentication might not be the core task.

Different scenarios call for different suitable algorithms:

| Scenario | With AES Acceleration | Without AES Acceleration |
|-------------------------|----------------------|-------------------------|
| Tamper detection needed | AES-GCM | ChaCha20-Poly1305 |
| Tamper detection not needed | AES-CTR | ChaCha20/XChaCha20 |

Multiple encryption methods can be provided for selection in the actual code.


### Configuration

Configuration options need to be added under `Config/Storage` to indicate whether encrypted storage is enabled and which encryption algorithm is selected. 
The `CryptoType` enum is used to represent the various algorithms.

```rust
// dragonfly-client-crypto/src/cryptor/crypto_type.rs
pub enum CryptoType {
    #[serde(rename = "chacha20-poly1305")]
    ChaCha20Poly1305,
    #[serde(rename = "aes-gcm")]
    AesGcm,
    #[serde(rename = "aes-ctr")]
    AesCtr,
    
}

// dragonfly-client-config/src/dfdaemon.rs
pub struct Storage {
    ......
    /// enable_encryption indicates whether to enable encryption for persistent cache storage.  
    #[serde(default = "default_storage_enable_encryption")]  
    pub enable_encryption: bool,

    /// encryption_algorithm indicates which algorithm will be used when encryption
    #[serde(default = "default_storage_encryption_algo")]
    pub encryption_algorithm: CryptoType,
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
  enableEncryption: true
  encryptionAlgorithm: chacha20-poly1305
```


### Metadata

The information required for encryption includes the plaintext, key, and nonce; the output is the ciphertext. 
For algorithms that support authentication, an authentication tag will also be generated.

For decryption, the required information includes the ciphertext, key, and nonce. 
For authenticated algorithms (AEAD), the tag is also necessary; the output is the corresponding plaintext.

The key, nonce, and tag used for decryption must match those used during encryption.

When encrypting different content with the same key, it is important to ensure that the nonce is different each time to enhance security.
If the same key and the same nonce are used to encrypt different content, an attacker may be able to deduce information about the plaintext from multiple ciphertexts.

A constructed nonce can be used to ensure uniqueness under the same key. 
For example, by combining part of the `task_id` with part of the `piece_number`, a unique nonce value can be generated. 
This approach eliminates the need to store a separate nonce for each piece.

Currently, the plan is to generate a key for each `PersistentCacheTask`, and all related `Pieces` will use this key for encryption and decryption. 
Since the algorithm used may change, the type of algorithm also needs to be recorded.

This information is stored in the metadata:

```rust
// dragonfly-client-storage/src/metadata.rs
pub struct PersistentCacheTask{
    ......
    /// crypto_info is the info saved for encryption/decryption
    pub crypto_info: Option<CryptoInfo>,
}

// dragonfly-client-crypto/src/cryptor/crypto_type.rs
pub struct CryptoInfo {
    // algorithm type
    pub crypto_type: CryptoType,
    pub key: Vec<u8>,
}
```

For authenticated algorithms (AEAD), each encrypted `Piece` needs to store its corresponding authentication tag:

```rust
// dragonfly-client-storage/src/metadata.rs
pub struct Piece {
    ......
    /// auth_tag is the tag generated by encryption algorithm when use AEAD
    pub auth_tag: Option<Vec<u8>>,
}
```


### Cryptor

The `PieceCryptor` trait defines the interface for the Cryptor:

```rust
// dragonfly-client-crypto/src/cryptor/piece_cryptor.rs
pub struct EncryptResult {
    pub ciphertext: Vec<u8>,
    pub tag: Option<Vec<u8>>,
}

pub trait PieceCryptor {
    fn encrypt_piece(&self, plaintext: &[u8], task_id: &str, piece_num: u32) -> Result<EncryptResult>;
    fn decrypt_piece(&self, ciphertext: &[u8], task_id: &str, piece_num: u32, tag: Option<&[u8]>) -> Result<Vec<u8>>;
    fn key_size() -> usize;
    fn nonce_size() -> usize;
    fn tag_size() -> Option<usize>;
}
```

`task_id` and `piece_num` are used to construct the nonce.

The `encrypt_piece` function returns an `EncryptResult`, which contains the encrypted `ciphertext` and an optional `tag` (an authentication tag is produced when using AEAD).

The `decrypt_piece` function returns the plaintext; its parameters include an optional `tag` (which is required for decryption when using AEAD).

The formats of the key, nonce, and other parameters differ between algorithms. For example, RustCrypto’s AES-CTR uses a 16-byte nonce, while AES-GCM uses a 12-byte nonce.

Therefore, interfaces like `key_size` and `nonce_size` are important for generating or constructing the corresponding information.

Below is an example implementation of `PieceCryptor` using the chacha20poly1305 algorithm:

```rust
pub struct ChaCha20Poly1305Cryptor {
    cipher: ChaCha20Poly1305,
}

impl ChaCha20Poly1305Cryptor {
    pub fn new(key: &[u8]) -> Self {
        let key = Key::from_slice(key);
        Self {
            // use RustCrypto crate
            cipher: ChaCha20Poly1305::new(key),
        }
    }

    // construct nonce from task_id and piece_number, it does not need to be loaded/saved from/to disk
    fn build_nonce(task_id: &str, piece_num: u32) -> Vec<u8> {
        let nonce_size = Self::nonce_size();
        let mut nonce = vec![0u8; nonce_size];
        let task_bytes = task_id.as_bytes();
        assert!(task_bytes.len() > 8);
        nonce[..8].copy_from_slice(&task_bytes[..8]); // nonce's first 8 bytes for task_id
        nonce[8..].copy_from_slice(&piece_num.to_be_bytes()); // remaining bytes for piece number
        assert!(nonce.len() == Self::nonce_size());
        nonce
    }

}

impl PieceCryptor for ChaCha20Poly1305Cryptor {
    fn encrypt_piece(&self, plaintext: &[u8], task_id: &str, piece_num: u32) -> Result<EncryptResult>{
        let nonce = Self::build_nonce(task_id, piece_num);
        let nonce = Nonce::from_slice(&nonce);
        
        // AEAD output will append auth tag after the ciphertext, so the output will be longer then plaintext
        // res is ciphertext + tag
        let res = self
            .cipher
            .encrypt(nonce, plaintext)
            .or_err(ErrorType::StorageError)?;
        
        // split res to ciphertext and tag,
        let tag_size = Self::tag_size().expect("should have tag size");
        let (ciphertext, tag) = res.split_at(res.len() - tag_size);
        Ok(EncryptResult { ciphertext: ciphertext.to_vec(), tag: Some(tag.to_vec()) })
    }

    fn decrypt_piece(&self, ciphertext: &[u8], task_id: &str, piece_num: u32, tag: Option<&[u8]>) -> Result<Vec<u8>>{
        let nonce = Self::build_nonce(task_id, piece_num);
        let nonce = Nonce::from_slice(&nonce);

        // concatenate ciphertext and tag for decryption
        let tag = tag.expect("should have tag");
        let mut combined = Vec::with_capacity(ciphertext.len() + tag.len());
        combined.extend_from_slice(ciphertext);
        combined.extend_from_slice(tag);
        let combined_slice: &[u8] = &combined;

        // decrypt
        let plaintext = self
            .cipher
            .decrypt(nonce, combined_slice)
            .or_err(ErrorType::StorageError)?;

        Ok(plaintext)
    }

    fn key_size() -> usize {
        32 // key size of chacha20-poly1305 is 32 bytes
    }

    fn nonce_size() -> usize {
        12 // nonce size of chacha20-poly1305 is 12 bytes
    }

    fn tag_size() -> Option<usize> {
        Some(16) // tag size of chacha20-poly1305 is 16 bytes
        // algorithm without authentication such as aes-ctr, this should return None
    }
}
```

`CryptorImpl` is the outermost wrapper for the encryptor/decryptor. 
By using an enum-based dispatch mechanism instead of `dyn`, it supports multiple encryption algorithm implementations:

```rust
// dragonfly-client-crypto/src/cryptor/piece_cryptor.rs
pub enum CryptorImpl {
    ChaCha20Poly1305(ChaCha20Poly1305Cryptor),
    // AesGcm(AesGcmCryptor),
    // AesCtr(AesCtrCryptor),
}

impl CryptorImpl {
    pub fn encrypt_piece(&self, plaintext: &[u8], task_id: &str, piece_num: u32) -> Result<EncryptResult> {
        match self {
            CryptorImpl::ChaCha20Poly1305(inner) => Ok(inner.encrypt_piece(plaintext, task_id, piece_num)?),
            // CryptorImpl::AesGcm(inner) => ...
        }
    }

    pub fn decrypt_piece(&self, ciphertext: &[u8], task_id: &str, piece_num: u32, tag: Option<&[u8]>) -> Result<Vec<u8>> {
        match self {
            CryptorImpl::ChaCha20Poly1305(inner) => Ok(inner.decrypt_piece(ciphertext, task_id, piece_num, tag)?),
            // CryptorImpl::AesGcm(inner) => ...
        }
    }
}
```

`CryptorImpl` is obtained via `get_cryptor`, where the key parameter is generated according to the `CryptoType`. 

Once the algorithm type is determined, the key format is also determined, so the key can be generated accordingly:

```rust
pub fn get_cryptor(crypto_type: &CryptoType, key: &[u8]) -> CryptorImpl {
    match crypto_type {
        // Select the specific encryption algorithm
        CryptoType::ChaCha20Poly1305 => {
            CryptorImpl::ChaCha20Poly1305(ChaCha20Poly1305Cryptor::new(key))
        }
        ......
        _ => unimplemented!("algorithm not supported"),
    }
}

impl CryptoType {
    // The algorithm type determines the format of the key/nonce/tag
    pub fn generate_key(&self) -> Vec<u8> {
        match self {
            // Generate the key according to the key format required by each algorithm
            CryptoType::ChaCha20Poly1305 => {
                // TODO random key
                vec![0u8; ChaCha20Poly1305Cryptor::key_size()]
            },
            ......
            _ => todo!(),
        }
    }
}
```

Encryption/Decryption example:

```rust
// encryption
let crypto_type = get_type_from_config_or_metadata(); // get algorithm used
let key = crypto_type.generate_key(); // key may need to be stored in task metadata
let cryptor = get_cryptor(&info.crypto_type, &info.key);
// get ciphertext, tag after encryption
let EncryptResult { ciphertext, tag } = cryptor.encrypt_piece_by_id(&plaintext, piece_id)?;

// decryption
let crypto_info = get_info_from_metadata();
let cryptor = get_cryptor(&info.crypto_type, &info.key);
// use key and tag(when use AEAD) to decrypt
let plaintext = cryptor.decrypt_piece_by_id(&ciphertext, piece_id, Some(tag)/None)?;
```


### Piece Encryption/Decryption

The main changes are concentrated in `dragonfly-client-storage/src/lib.rs` and `dragonfly-client-storage/src/content.rs`.

Taking the download process as an example, after entering `dfdaemon_download.rs/download_persistent_cache_task`, 
the function `storage.download_persistent_cache_task_started` will be called first.

In this function, encryption is enabled or disabled based on the configuration, and the encryption information is stored in the task metadata.

```rust
// dragonfly-client-storage/src/lib.rs
pub async fn download_persistent_cache_task_started(
        &self, args...
    )-> Result<metadata::PersistentCacheTask> {

    // ADDED do encryption when config enable is set
    let crypto_info = if self.config.storage.enable_encryption {
        let crypto_type = self.config.storage.encryption_algorithm.clone();
        let key = crypto_type.generate_key();
        Some(CryptoInfo{crypto_type: crypto_type, key: key})
    } else {
        None
    };
    
    let metadata = self.metadata.download_persistent_cache_task_started(
        id,
        ttl,
        persistent,
        piece_length,
        content_length,
        created_at,
        // ADDED store key/algorithm type in the metadata of task
        crypto_info,
    )?;

    self.content
        .create_persistent_cache_task(id, content_length)
        .await?;
    Ok(metadata)
}
```

The subsequent execution flow is as follows:

```
task_manager_clone.download()
    PersistentCacheTask.download_partial_from_local()
    if (need_piece_content) piece.download_persistent_cache_from_local_into_async_read()
            storage.upload_persistent_cache_piece()
                content.read_persistent_cache_piece()
```

The program will check for any existing local cache. When `need_piece_content` is set, it will read content from the local cache.

Therefore, decryption needs to be handled here. If the task metadata read by the upper layer contains encryption information, 
the necessary decryption parameters should be passed in as arguments.

```rust
// dragonfly-client-storage/src/content.rs
pub async fn read_persistent_cache_piece(
    &self,
    task_id: &str,
    offset: u64,
    length: u64,
    range: Option<Range>,
    // ADDED
    piece_id: &str,
    crypto_info: Option<&CryptoInfo>,
    auth_tag: Option<&[u8]>,
) -> Result<impl AsyncRead> {
    // original code ↓
    let task_path = self.get_persistent_cache_task_path(task_id);

    let (target_offset, target_length) = calculate_piece_range(offset, length, range);

    let f = File::open(task_path.as_path()).await.inspect_err(|err| {
        error!("open {:?} failed: {}", task_path, err);
    })?;
    let mut f_reader = BufReader::with_capacity(self.config.storage.read_buffer_size, f);

    f_reader
        .seek(SeekFrom::Start(target_offset))
        .await
        .inspect_err(|err| {
            error!("seek {:?} failed: {}", task_path, err);
        })?;
    // original code ↑

    // ADDED
    let res = if let Some(info) = crypto_info {
        // Read ciphertext from file
        let mut ciphertext = vec![0u8; target_length as usize];
        f_reader.read_exact(&mut ciphertext).await?;

        // decryption
        let cryptor = get_cryptor(&info.crypto_type, &info.key);
        let plaintext = cryptor.decrypt_piece_by_id(&ciphertext, piece_id, auth_tag)?;

        Either::Left(Cursor::new(plaintext))
    } else {
        Either::Right(f_reader.take(target_length))
    };

    Ok(res)
}
```

The following is the subsequent call process:

```
task_manager_clone.download()
    PersistentCacheTask.download_partial_with_scheduler()
        PersistentCacheTask.download_partial_with_scheduler_from_parent()
            download_from_parent()
                piece_manager.download_persistent_cache_from_parent()
                    storage.download_persistent_cache_piece_started()
                    downloader.download_persistent_cache_piece()
                    storage.download_persistent_cache_piece_from_parent_finished()
                        content.write_persistent_cache_piece()
```

The downloaded content is then passed to `content.write_persistent_cache_piece()`, which is responsible for writing it to the file. 
Therefore, encryption logic needs to be added here.

During the encryption process, the CRC should be calculated based on the original plaintext, 
and the return value should include the tag to pass the AEAD authentication tag.

```rust
// dragonfly-client-storage/src/content.rs
pub async fn write_persistent_cache_piece<R: AsyncRead + Unpin + ?Sized>(
    &self,
    task_id: &str,
    offset: u64,
    expected_length: u64,
    reader: &mut R,
    // ADDED
    piece_id: &str,
    crypto_info: Option<&CryptoInfo>,
) -> Result<WritePieceResponse> {
    // original code ↓
    // Open the file and seek to the offset.
    let task_path = self.get_persistent_cache_task_path(task_id);
    let mut f = OpenOptions::new()
        .truncate(false)
        .write(true)
        .open(task_path.as_path())
        .await
        .inspect_err(|err| {
            error!("open {:?} failed: {}", task_path, err);
        })?;

    f.seek(SeekFrom::Start(offset)).await.inspect_err(|err| {
        error!("seek {:?} failed: {}", task_path, err);
    })?;
    // original code ↑

    // ADDED need encrypt
    if let Some(info) = crypto_info {
        // 1. read plaintext
        let mut plaintext = Vec::new();
        reader.read_to_end(&mut plaintext).await?;

        // 2. use plaintext to calculate crc
        let mut hasher = crc32fast::Hasher::new();
        hasher.update(&plaintext);
        let hash = hasher.finalize().to_string();

        // 3. encrypt
        let cryptor = get_cryptor(&info.crypto_type, &info.key);
        let EncryptResult { ciphertext, tag } = cryptor.encrypt_piece_by_id(&plaintext, piece_id)?;

        // 4. write
        let mut writer = BufWriter::with_capacity(self.config.storage.write_buffer_size, f);
        writer.write_all(&ciphertext).await?;
        writer.flush().await?;

        // check length
        if ciphertext.len() as u64 != expected_length {
            return Err(Error::Unknown(format!(
                "expected length {} but got {}",
                expected_length, ciphertext.len()
            )));
        }

        Ok(WritePieceResponse {
            length: ciphertext.len() as u64,
            hash: hash,
            // ADDED
            auth_tag: tag,
        })

    } else {
        // if don not need encryption, do original process
        ......
        Ok(WritePieceResponse {
            length,
            hash: hasher.finalize().to_string(),
            // ADDED
            auth_tag: None
        })
    }
}
```

---

The upload process reuses many of the same `Storage` calls as the download process, such as `content.write_persistent_cache_piece`, and similar modifications can be made to the differing parts:

```rust
// dragonfly-client-storage/src/lib.rs
// This is unique to the upload process.
pub async fn create_persistent_cache_task_started(
    &self, ...
) -> Result<metadata::PersistentCacheTask> {
    // ADDED whether need to encrypt
    let info = if self.config.storage.enable_encryption {
        let crypto_type = self.config.storage.encryption_algorithm.clone();
        let key = crypto_type.generate_key();
        Some(CryptoInfo{crypto_type: crypto_type, key: key})
    } else {
        None
    };

    let metadata = self.metadata.create_persistent_cache_task_started(
        id,
        ttl,
        piece_length,
        content_length,
        // ADDED store crypto_info in metadata
        info,
    )?;

    self.content
        .create_persistent_cache_task(id, content_length)
        .await?;
    Ok(metadata)
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