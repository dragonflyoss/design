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

```
dragonfly-client-storage/src/
└── encrypt/
    └── cryptor.rs    # Crypt implementation
```
* `piece.rs`: Add data encryption/decryption
* `storage/src/lib.rs`: Add key generation for Task
* `metadata.rs`: Add key or key_id in Task metadata


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

However, on platforms without hardware support, its performance might not be as good as ChaCha20.

Besides encryption, authentication can check if data has been maliciously tampered with, but at the cost of needing to store additional authentication information. AES-GCM and ChaCha20-Poly1305 have this capability.


Dragonfly's current design writes Pieces to their corresponding offsets in the final file. This implies that each Piece must be encrypted prior to being written to disk to ensure no plaintext information is stored during the entire process.


In this case, authenticated algorithms would introduce additional complexity because authentication information would need to be saved for each encrypted Piece block.

Given that Dragonfly is typically deployed on servers, it is highly likely to have AES hardware acceleration, so AES speed should be optimal. 

At the same time, Dragonfly uses CRC32 and other verification methods, which to some extent defend against tampering (but not completely). Authentication might not be the core task, so AES-CTR could be be a reasonable choice.

Multiple encryption methods can be provided for selection in the actual code.


### Cryptor

The `Cryptor` can receive multiple `CryptoAlgo` implementations and use them for encryption and decryption. 
This allows for supporting different encryption algorithms by writing various implementations.

```rust
/// Encryption algorithm trait
pub trait CryptoAlgo {
    fn encrypt(&self, plaintext: Vec<u8>) -> Result<Vec<u8>>;
    fn decrypt(&self, ciphertext: Vec<u8>) -> Result<Vec<u8>>;
    ......
}

pub struct Cryptor<A: CryptoAlgo> {
    algo: A,
}

impl<A: CryptoAlgo> Cryptor<A> {
    pub fn new(algo: A) -> Self {
        Self { algo }
    }

    pub fn encrypt_piece<R: Read, W: Write>(&self, mut reader: R, mut writer: W) -> Result<()> {
        let ciphertext = self.algo.encrypt(...)?;
        ......
        Ok(())
    }

    pub fn decrypt_file<R: Read, W: Write>(&self, mut reader: R, mut writer: W) -> Result<()> {
        let plain = self.algo.decrypt(...)?;
        ......
        Ok(())
    }
}

/// AES-CTR implement
pub mod aes_ctr_impl {
    use super::*;
    use aes::Aes256;
    use ctr::cipher::{KeyIvInit, StreamCipher, StreamCipherSeek};
    use ctr::Ctr128BE;

    pub struct AesCtrAlgo {
        key: [u8; 32],
        base_iv: [u8; 16],
    }

    impl AesCtrAlgo {
        pub fn new(key: [u8; 32], base_iv: [u8; 16]) -> Self {
            Self { key, base_iv }
        }
    }

    impl CryptoAlgo for AesCtrAlgo {
        fn encrypt(&self, plaintext: Vec<u8>) -> Result<Vec<u8>> {
            let mut cipher = Ctr128BE::<Aes256>::new(&self.key.into(), &self.base_iv.into())
            cipher.apply_keystream(&mut plaintext);
            Ok(buf)
        }

        fn decrypt(&self, ciphertext: Vec<u8>) -> Result<Vec<u8>> {
            self.encrypt(ciphertext)
        }
    }
}

```

Use of different encryption algorithms:

```rust
// AES-CTR
let (key, iv) = gen_aes_iv();
let algo = AesCtrAlgo::new(key, iv);
let aes_cryptor = Cryptor::new(algo);

aes_cryptor.encrypt_piece(...)
aes_cryptor.decrypt_piece(...)


// ChaCha20
let (key, nonce) = gen_key_nonce();
let algo = ChaChaAlgo::new(&key, nonce);
let chacha_cryptor = Cryptor::new(algo);

chacha_cryptor.encrypt_piece(...)
chacha_cryptor.decrypt_piece(...)
```




### PersistentCacheTask

For each `Task` that requires encryption, a corresponding key is generated and saved in the Task's `metadata`. 
Key generation should be completed before new `metadata` is created.

```rust
/// New member for key
pub struct PersistentCacheTask {
    ...
	// TODO
    pub entrypt_id_or_key: Option<String>,
}

/// Generate key when task started 
pub async fn download_persistent_cache_task_started(...) -> Result<metadata::PersistentCacheTask> {
	// ADD TODO
	let key = generate_key();

	let metadata = self.metadata.download_persistent_cache_task_started(
		id,
		ttl,
		persistent,
		piece_length,
		content_length,
		created_at,
		//ADD TODO
		key,
	)?;
	......
}
```



### Piece Write/Read Operations

* **Writing**: Before a piece of data is written to the persistent cache, the `cryptor` will encrypt it. The encrypted piece will then be stored.

    For example, when downloading a piece:

    ```rust
    // dragonfly-client/src/resource/piece.rs

    let (content, offset, digest) = self
        .downloader
        .download_persistent_cache_piece(
            ...
        )
        .await
        .inspect_err(|err| {
            ...
        })?;

    // ADD TODO
    let encrypted_piece = cryptor.entrypt_piece(content).unwarp();
    let mut reader = Cursor::new(encrypted_piece);

    // Record the finish of downloading piece.
    match self
        .storage
        .download_persistent_cache_piece_from_parent_finished(
            piece_id,
            task_id,
            offset,
            length,
            digest.as_str(),
            parent.id.as_str(),
            &mut reader,
        )
        .await
    {
        ...
    }
    ```



* **Reading**: When a piece of data is requested from the persistent cache, the encrypted piece will be retrieved, and the `cryptor` will decrypt it before it's made available.
    
    For example, when uploading a piece:
    
    ```rust
    // dragonfly-client-storage/src/lib.rs
    pub async fn upload_piece(...) {
        ......
        match self.metadata.get_piece(piece_id) {
            Ok(Some(piece)) => {
                ......
                match self
                    .content
                    .read_piece(task_id, piece.offset, piece.length, range)
                    .await
                {
                    Ok(reader) => {
                        // ADD TODO
                        reader = cryptor.decrypt_piece(...);
                        // Finish uploading the task.
                        self.metadata.upload_task_finished(task_id)?;
                        Ok(Either::Right(reader))
                    }
                    ......
                }
            }
            ......
        }
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