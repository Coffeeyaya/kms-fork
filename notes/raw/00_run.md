# Running Zama KMS

Based on the documentation in [kms-docker-blocked-private-images.md](./kms-docker-blocked-private-images.md), **Docker compose/builds do not work out of the box for external contributors**. The official configurations attempt to pull private, authenticated base images from Zama's registries (`ghcr.io/zama-ai/kms/rust-golden-image:latest`) and Chainguard (`cgr.dev/zama.ai/glibc-dynamic`).

To run the system locally, external contributors **must use the Manual Native approach** via `cargo`.

---

## 1. Centralized Case (Single-Node)

The centralized mode runs a single KMS core. This is the simplest configuration to get started.

### Manual / Native (Verified Working)

This compiles and runs the services directly on your local machine.

#### Step 1: Install System Prerequisites
Ensure you have the required build tools and libraries installed:
```bash
# MacOS (Homebrew)
brew install protobuf openssl pkg-config

# Ubuntu/Debian
sudo apt-get update && sudo apt-get install -y protobuf-compiler pkg-config libssl-dev build-essential
```

#### Step 2: Compile Binaries
Build the key generator, server, and client binaries from source:
```bash
cargo build --bin kms-gen-keys --bin kms-server --bin kms-core-client
```

#### Step 3: Generate Server Signing Keys
Before starting the server, you must generate the server's signing keys and verification certificates. Run the following:
```bash
cargo run --bin kms-gen-keys -- centralized
```
This generates the signing key material under `./keys/PRIV/` and verification keys under `./keys/PUB/`.

#### Step 4: Run the Server
Start the standalone centralized server. It will read the generated signing keys and listen for gRPC requests on port `50051`:
```bash
cargo run --bin kms-server -- --config-file core/service/config/default_centralized.toml
```

#### Step 5: Run Client Operations (Separate Terminal)
Open a new terminal tab. To perform operations, you must first run a **preprocessing** step to generate a preprocessing ID, and then run **key generation**:

1. **Run Preprocessing:**
   ```bash
   cargo run --bin kms-core-client -- -f core-client/config/client_local_centralized.toml insecure-preproc-key-gen
   ```
   Copy the printed `request_id` (this is your `<PREPROC_ID>`).

2. **Run Key Generation:**
   ```bash
   cargo run --bin kms-core-client -- -f core-client/config/client_local_centralized.toml insecure-key-gen --preproc-id <PREPROC_ID>
   ```
   This creates the FHE keys under the `./keys/` directory.

---

## 2. Threshold Case (Multi-Node Cluster)

In the threshold setup, 4 independent server nodes collaborate to generate a shared key using Multi-Party Computation (MPC).

### Manual / Native running
To run the threshold cluster natively on your machine, you must spin up all 4 servers.

> [!IMPORTANT]
> The default configuration TOMLs (`compose_1.toml` through `compose_4.toml`) use hostnames like `abcd.dev-kms-core-1.com` to communicate. To run natively, you must either:
> 1. Map these hostnames to `127.0.0.1` in your `/etc/hosts` file:
>    ```text
>    127.0.0.1 abcd.dev-kms-core-1.com abcd.dev-kms-core-2.com abcd.dev-kms-core-3.com abcd.dev-kms-core-4.com
>    ```
> 2. Or modify the `[[threshold.peers]]` sections in the config TOMLs to use `localhost` or `127.0.0.1`.
> Additionally, because they write public/private data, you must have an S3 compatible storage running (e.g. MinIO) or adapt the configurations to use local file storage.

#### Step 1: Compile Binaries
```bash
cargo build --bin kms-gen-keys --bin kms-server --bin kms-core-client
```

#### Step 2: Generate Signing Keys and Certificates
Generate the signing keys and mTLS certificates for all 4 parties (this is typically automated in Docker, but must be run for each party ID natively):
```bash
for i in 1 2 3 4; do \
  cargo run --bin kms-gen-keys -- \
    --private-storage file --private-file-path ./keys \
    --public-storage file --public-file-path ./keys \
    threshold --signing-key-party-id $i --tls-subject dev-kms-core-$i.com; \
done
```

#### Step 3: Run the Nodes (4 Terminals)
Open 4 separate terminal windows and run one node command in each:
- **Node 1:** `cargo run --bin kms-server -- --config-file core/service/config/compose_1.toml`
- **Node 2:** `cargo run --bin kms-server -- --config-file core/service/config/compose_2.toml`
- **Node 3:** `cargo run --bin kms-server -- --config-file core/service/config/compose_3.toml`
- **Node 4:** `cargo run --bin kms-server -- --config-file core/service/config/compose_4.toml`

#### Step 4: Run Client Operations (5th Terminal)
Open a 5th terminal window and run:

1. **Run Preprocessing:**
   ```bash
   cargo run --bin kms-core-client -- -f core-client/config/client_local_threshold.toml insecure-preproc-key-gen
   ```
   Copy the printed `<PREPROC_ID>`.

2. **Run Key Generation:**
   ```bash
   cargo run --bin kms-core-client -- -f core-client/config/client_local_threshold.toml insecure-key-gen --preproc-id <PREPROC_ID>
   ```

---

## 3. Docker Compose (Zama Org Members Only)

If you are a Zama org member or have access to the private registries, you can run the services using Docker Compose.

> [!WARNING]
> This approach will fail for external contributors due to `403 Forbidden` / `401 Unauthorized` errors when fetching base images.

### Centralized Mode
```bash
docker compose -f docker-compose-core-base.yml -f docker-compose-core-centralized.yml up --build
```

### Threshold Mode
```bash
docker compose -f docker-compose-core-base.yml -f docker-compose-core-threshold.yml up --build
```

To run with custodian-based backup enabled:
```bash
KMS_DOCKER_BACKUP_SECRET_SHARING=true \
  docker compose -f docker-compose-core-base.yml -f docker-compose-core-threshold.yml up --build
```