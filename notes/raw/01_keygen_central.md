### Focus Areas for Centralized Key Generation

To follow the journey of this data structure and witness how a key is actually created, you should focus on these files:

1. **📂 Client Handler:** [core-client/src/keygen.rs](file:///Users/tsaiyunchen/Desktop/CSLAB/04_Work/kms/core-client/src/keygen.rs)
    - **What to look for:** This is the client-side entry point. When you trigger client operations, the client first runs a preprocessing query (`insecure-preproc-key-gen` / `preproc-key-gen`) to obtain a preprocessing ID. It then packages a key generation request (`insecure-key-gen` / `key-gen`) passing the preprocessing ID, signs the request using EIP-712 schemas to authorize the action, and dispatches it.
        
2. **📂 Server Endpoint:** [core/service/src/engine/centralized/service/key_gen.rs](file:///Users/tsaiyunchen/Desktop/CSLAB/04_Work/kms/core/service/src/engine/centralized/service/key_gen.rs)
    - **What to look for:** This is the server-side gRPC entry point (`key_gen_impl`). The server receives the signed request, parses and validates it, checks rate limits and idempotency, and immediately spawns a background task (`key_gen_background`) so the client gets an instant response.
        
3. **📂 Cryptographic Core:** [core/service/src/engine/centralized/central_kms.rs](file:///Users/tsaiyunchen/Desktop/CSLAB/04_Work/kms/core/service/src/engine/centralized/central_kms.rs)
    - **What to look for:** This contains the core centralized keygen algorithms (`generate_uncompressed_fhe_keys` / `generate_fhe_keys`). Because key generation is extremely CPU-heavy, it offloads calculations from Tokio to a separate CPU Rayon thread pool (`rayon::spawn_fifo`). It calls `ClientKey::generate` from `tfhe-rs` to sample the secret key `s`, derives the public evaluation keys (like the bootstrapping key), and constructs the key digests.
        
4. **📂 Storage Serialization:** [core/service/src/vault/storage/crypto_material/centralized.rs](file:///Users/tsaiyunchen/Desktop/CSLAB/04_Work/kms/core/service/src/vault/storage/crypto_material/centralized.rs)
    - **What to look for:** Once generated, the keys are serialized and persisted to disk. This script translates live Rust memory objects into binary payloads and writes them under the directory configured in your server TOML (by default `./keys/`).

---

# TOML files for server and client

## 1. Server Configuration: [default_centralized.toml](file:///Users/tsaiyunchen/Desktop/CSLAB/04_Work/kms/core/service/config/default_centralized.toml)

This file sets up the **standalone single server node** that acts as the centralized manager.

### `[service]` (Network Entryway)
- **`listen_address = "0.0.0.0"`**: Tells the server to listen to incoming network connections from any IP.
- **`listen_port = 50051`**: The port number assigned to this server.
- **`grpc_max_message_size = 104857600`**: Allows massive FHE keys (up to 100 MiB) to be transferred over the API.

### `[public_vault.storage.file]` & `[private_vault.storage.file]`
- **`path = "./keys"`**: Tells the server to write vault directories inside `./keys`. In centralized mode, public keys go to `./keys/PUB/` and private keys to `./keys/PRIV/`.

### `[rate_limiter_conf]`
- Allocates operation request weights. Key generation is expensive (`keygen = 1000`), whereas decryption is cheap (`pub_decrypt = 1`).

---

## 2. Client Configuration: [client_local_centralized.toml](file:///Users/tsaiyunchen/Desktop/CSLAB/04_Work/kms/core-client/config/client_local_centralized.toml)

This file tells the **client application** how to target and sign its requests.

### Global System Flags
- **`kms_type = "centralized"`**: Bypasses threshold MPC coordination layers.
- **`num_parties = 1` / `num_majority = 1` / `num_reconstruct = 1`**: central mode quorums.
- **`fhe_params = "Test"`**: Uses lightweight, fast parameters for development.

### `[[cores]]`
- **`party_id = 1`** and **`address = "localhost:50051"`**: Targets the centralized server port.

### `[default_domain]`
- Configuration for EIP-712 standard web3 signatures to prove client authenticity.

---

## How to Run This Phase Locally

To perform key generation natively:

1. **Generate Server Signing Keys:**
    ```bash
    cargo run --bin kms-gen-keys -- centralized
    ```
2. **Launch the Server:**
    ```bash
    cargo run --bin kms-server -- --config-file core/service/config/default_centralized.toml
    ```
3. **Trigger Preprocessing (In a second terminal):**
    ```bash
    cargo run --bin kms-core-client -- -f core-client/config/client_local_centralized.toml insecure-preproc-key-gen
    ```
    This prints a `request_id` (your `<PREPROC_ID>`).
4. **Trigger Key Generation:**
    ```bash
    cargo run --bin kms-core-client -- -f core-client/config/client_local_centralized.toml insecure-key-gen --preproc-id <PREPROC_ID>
    ```
    The server will execute the cryptographic logic and write keys to `./keys/`.
