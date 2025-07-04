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
└── entrypt/
    └── entryptor.rs    # Entryptor implementation
```
* `piece.rs`: Add data encryption/decryption
* `storage/src/lib.rs`: Add key generation for Task
* `metadata.rs`: Add key or key_id in Task metadata


## Implementation

### Encryptor

```rust
/// Encryption algorithm trait
pub trait CryptoAlgo {
    fn encrypt(&self, plaintext: Vec<u8>) -> Result<Vec<u8>>;
    fn decrypt(&self, ciphertext: Vec<u8>) -> Result<Vec<u8>>;
    ......
}

pub struct Encryptor<A: CryptoAlgo> {
    algo: A,
}

impl<A: CryptoAlgo> Encryptor<A> {
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

### PersistentCacheTask

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

  * **Writing**: Before a piece of data is written to the persistent cache, the `entryptor` will encrypt it. The encrypted piece will then be stored.
  * **Reading**: When a piece of data is requested from the persistent cache, the encrypted piece will be retrieved, and the `entryptor` will decrypt it before it's made available.



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