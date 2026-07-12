# KMS: Native Centralized Mode (Codespace)

> Run KMS server + client entirely on GitHub Codespace from source — no Docker needed.  
> **Status: ✅ Verified working** (see execution logs below).

---

## Architecture

```
[GitHub Codespace]
  ├── kms-server    (centralized, port 50051)
  └── kms-core-client  (talks to server on localhost:50051)
```

Keys stored locally at `file:///workspaces/kms/keys`.

---

## Step 1: Environment Setup

```bash
# Install Rust (if not present)
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh -s -- -y
source "$HOME/.cargo/env"

# Install system dependencies
sudo apt-get update && sudo apt-get install -y \
  protobuf-compiler pkg-config libssl-dev build-essential

# Add swap to prevent OOM during compilation (heavy crypto crates)
sudo fallocate -l 8G /tmp/swapfile
sudo chmod 600 /tmp/swapfile
sudo mkswap /tmp/swapfile
sudo swapon /tmp/swapfile
# Verify: free -h
```

---

## Step 2: Build Binaries

```bash
cd /workspaces/kms
cargo build --bin kms-gen-keys --bin kms-server --bin kms-core-client
```

> Takes ~15–30 min on first build. Subsequent builds are fast.

---

## Step 3: Generate Signing Keys

```bash
cd /workspaces/kms
cargo run --bin kms-gen-keys -- --config-file core/service/config/keygen_centralized.toml
```

Keys are written to:
- `core/service/keys/PUB`
- `core/service/keys/PRIV`

---

## Step 4: Start the Server

```bash
cd /workspaces/kms/core/service
cargo run --bin kms-server -- --config-file config/default_centralized.toml
```

Server listens on `0.0.0.0:50051`.

**Verify in a new terminal:**
```bash
ss -ltnp | grep 50051

# Or health check:
cd /workspaces/kms
cargo run -p kms-health-check -- live --endpoint localhost:50051
```

Expected: `Reachable` (may show `Degraded` until FHE keys are generated in Step 5).

---

## Step 5: Run Client Operations

All commands from `/workspaces/kms` in a new terminal tab.

### 5a. Preprocessing
```bash
cargo run --bin kms-core-client -- \
  -f core-client/config/client_local_centralized.toml \
  insecure-preproc-key-gen
```
→ Copy the printed `request_id` as `<PREPROC_ID>`

### 5b. Key Generation
```bash
cargo run --bin kms-core-client -- \
  -f core-client/config/client_local_centralized.toml \
  insecure-key-gen --preproc-id <PREPROC_ID>
```
→ Copy the printed `request_id` as `<KEY_ID>`

### 5c. Encrypt
```bash
cargo run --bin kms-core-client -- \
  -f core-client/config/client_local_centralized.toml \
  encrypt \
  --to-encrypt 0500000000000000 \
  --data-type euint64 \
  --key-id <KEY_ID> \
  --ciphertext-output-path ./ciphertext.bin
```

### 5d. Decrypt & Verify
```bash
cargo run --bin kms-core-client -- \
  -f core-client/config/client_local_centralized.toml \
  public-decrypt from-file \
  --input-path ./ciphertext.bin
```

✅ Expected: `plaintexts: [TypedPlaintext { bytes: [5, 0, 0, 0, 0, 0, 0, 0], fhe_type: 5 }]`

---

## Execution Logs (Real Run)

```
$ cargo run --bin kms-core-client -- -f core-client/config/client_local_centralized.toml insecure-preproc-key-gen
insecure preproc done - "request_id": "392eb262fd6b43b0f29f86ce028f5b49d4331f3285c8d35853b5ee417f6596bf"

$ cargo run --bin kms-core-client -- -f core-client/config/client_local_centralized.toml insecure-key-gen --preproc-id 392eb262...
insecure keygen done - "request_id": "433293f5a364b07844b28b450d5e51f4b5842f0baeaa9d153006a5a7b4ad30f4"

$ cargo run --bin kms-core-client -- ... encrypt --key-id 433293f5... --ciphertext-output-path ./ciphertext.bin
Encryption generated - no request_id returned

$ cargo run --bin kms-core-client -- ... public-decrypt from-file --input-path ./ciphertext.bin
plaintexts: [TypedPlaintext { bytes: [5, 0, 0, 0, 0, 0, 0, 0], fhe_type: 5 }]
```

---

## What Each Step Does

| Step | What happens |
|---|---|
| Preprocessing | Server pre-computes cryptographic randomness, returns a `PREPROC_ID` |
| Key Gen | Server uses preprocessing to generate FHE public + secret keys, returns `KEY_ID` |
| Encrypt | **Client-side only** — loads public key, encrypts locally, writes `ciphertext.bin` |
| Decrypt | Client sends ciphertext to server → server decrypts with secret key → returns plaintext + EIP-712 signature |

---

## Troubleshooting

| Symptom | Fix |
|---|---|
| OOM during `cargo build` | Ensure 8 GB swap is active (`free -h`) |
| `connection refused :50051` | Server not started yet |
| `Operator Key: Not available` | Generate keys (Step 3) before starting server |
| Client config s3 errors | Ensure `s3_endpoint = "file:///workspaces/kms/keys"` in `client_local_centralized.toml` |
