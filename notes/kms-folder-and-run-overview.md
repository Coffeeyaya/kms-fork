# KMS Overview

> Zama KMS — Threshold Key Management System for TFHE (FHE).  
> Repo: `04_Work/kms/`

**Related notes:**
- [LWE Math → Code Mapping](./lwe-math-to-code-mapping.md)
- [Key Generation Deep Dive](./kms-keygen-deep-dive.md)

---

## What is KMS?

A fully decentralized key management solution for TFHE, using a maliciously secure MPC protocol (secret sharing) for:
- Threshold key generation
- Threshold decryption of TFHE ciphertexts
- Key resharing
- Distributed CRS setup for ZK proofs

Interaction: via **gRPC** or through **FHEVM Gateway**.

Two deployment modes:
- **Centralized** — single core, plain operations (testing / secure hardware only)
- **Threshold** — 4+ MPC parties running secure protocols (default)

---

## Folder Structure

| Folder | Purpose |
|---|---|
| `core/` | Core Rust crate — crypto primitives & MPC protocols (keygen, decryption, CRS) |
| `core-client/` | CLI tool to interact with KMS gRPC endpoints; also used for tests |
| `bc2wrap/` | Blockchain-to-KMS wrapper/adapter layer |
| `tools/` | Utility binaries (e.g. `kms-custodian` for backup/recovery) |
| `docker/` | Docker image build configs |
| `charts/` | Helm/K8s deployment charts |
| `ci/` | CI/CD pipeline scripts |
| `docs/` | Full documentation (guides, tutorials, references, getting-started) |
| `observability/` | Monitoring configs (Prometheus, Jaeger) |
| `backward-compatibility/` | Tests/tools to ensure API/protocol compat across versions |
| `ai-docs/` | AI-oriented documentation |

---

## Focused Reading Guide (Server + Client Implementation)

### Server
| Priority | Folder | Focus |
|---|---|---|
| 1 | `core/` | MPC protocols, threshold keygen/decryption, crypto logic |
| 2 | `core/service/` | gRPC service layer — request handling |
| 3 | `core/service/config/` | Per-party server config TOMLs |

### Client
| Priority | Folder | Focus |
|---|---|---|
| 1 | `core-client/` | CLI client — all commands (keygen, CRS, decrypt, backup) |
| 2 | `core-client/config/` | Client config TOMLs (threshold vs centralized) |
| 3 | `core-client/src/` | Source — how gRPC calls are constructed |

### Supporting Context
| Folder | Why |
|---|---|
| `docs/guides/core_client.md` | Full operation reference |
| `docs/getting-started/` | Architecture overview |

**Skip**: `ci/`, `charts/`, `observability/`, `backward-compatibility/`, `ai-docs/`, `bc2wrap/`, `tools/`

---

## How to Run

### Prerequisites
- Docker (>= 24 GB RAM allocated)
- Rust + `rustup` (version auto-picked from `rust-toolchain.toml`)
- `protoc`, `pkgconfig`, `openssl`

### Option 1: Native (Rust)
```bash
cargo build
cargo test -F testing --lib   # unit tests only (no Redis)
cargo test -F testing         # integration tests (needs Redis)
```

### Option 2: Docker (Recommended for local dev)

#### Step 0 — Authenticate to `ghcr.io` (private images)
```bash
# Generate a GitHub PAT with "read:packages" scope at:
# https://github.com/settings/tokens  (Tokens classic)

docker login ghcr.io
# Username: <your-github-username>
# Password: <your-PAT>
```

> Token is saved by Docker in the clear — use minimal scope + short expiry.

#### Step 1 — Check `.env` at repo root
```env
MINIO_ROOT_USER=admin
MINIO_ROOT_PASSWORD=strongadminpassword
```

#### Step 2 — Build & Run

**Threshold (4 nodes, default):**
```bash
docker compose -vvv -f docker-compose-core-base.yml -f docker-compose-core-threshold.yml build
docker compose -vvv -f docker-compose-core-base.yml -f docker-compose-core-threshold.yml up
```

**With custodian-based backup:**
```bash
KMS_DOCKER_BACKUP_SECRET_SHARING=true \
  docker compose -vvv -f docker-compose-core-base.yml -f docker-compose-core-threshold.yml up
```

**Centralized (single node, simpler):**
```bash
docker compose -vvv -f docker-compose-core-base.yml -f docker-compose-core-centralized.yml up
```

#### Step 3 — Interact via core-client

**Native (from `core-client/`):**
```bash
cargo run -- -f config/client_local_threshold.toml crs-gen
cargo run -- --help
```

**Via Docker image:**
```bash
docker pull ghcr.io/zama-ai/kms/core-client:latest

docker run --network host -v ./core-client/config:/config \
  ghcr.io/zama-ai/kms/core-client:latest \
  kms-core-client -f /config/client_local_threshold.toml --help
```

---

## Docker Compose Architecture

Compose files are layered:
- `docker-compose-core-base.yml` — S3 mock (MinIO) base
- `+ docker-compose-core-centralized.yml` — single `dev-kms-core`
- `+ docker-compose-core-threshold.yml` — 4 MPC parties
- `+ docker-compose-core-threshold-6.yml` — 6 MPC parties
- `+ docker-compose-telemetry.yml` — Prometheus + Jaeger sidecars

Startup dependency order (threshold):
```
dev-s3-mock --> dev-s3-mock-setup --> dev-kms-core-gen-signing-keys-ca-certs
                                  --> dev-kms-core-1/2/3/4 --> dev-kms-core-init
```

---

## Key References
- [core_client.md](../kms/docs/guides/core_client.md) — Full CLI reference
- [README.md](../kms/README.md) — Project overview
- [Noah's Ark paper](https://eprint.iacr.org/2023/815) — MPC protocol spec

## Notes in This Folder
- [LWE Math → Code Mapping](./lwe-math-to-code-mapping.md)
- [Key Generation Deep Dive](./kms-keygen-deep-dive.md)
