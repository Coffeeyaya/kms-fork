← [Back to Workspace Index](../index.md)

# Prism Phase: OPRF & KMS Integration Learning Map

I am building the cryptographic boundary of the `prism` project: the **OPRF Tokenization Service** and the **Threshold KMS Client**. Other engineers are building the frontend, graph compositors, and databases. My role is to build the secure interfaces they will consume.

Here is my engineering context:
1. **Target Stack**: Rust, `axum` (for the OPRF REST API), `tonic`/`prost` (for the external Zama KMS gRPC interactions), and `tokio`.
2. **Infrastructure Environment**: Docker Compose orchestrating Redpanda (Kafka), PostgreSQL, and MinIO locally.
3. **Reference Repository**: Zama's `kms` repository (`00_Reference/repos/kms`), specifically the `core-client/` implementation.

> **Run guides moved to dedicated notes:**
> - Centralized: [kms-run-centralized-on-codespace.md](./kms-run-centralized-on-codespace.md)
> - Docker (blocked): [kms-docker-blocked-private-images.md](./kms-docker-blocked-private-images.md)

---

## Execution plan and learning map

### Phase 1: Cryptographic Primitives (OPRF Math & Memory Safety)
- **Boundary Clarification:** Zama KMS *does not* provide OPRF. I must implement this entirely from scratch.
- **Goal:** Implement the raw cryptographic logic before dealing with networks or servers.
- **Key Topics:**
  - VOPRF Standardization (IETF RFC 9497): Implementing NIST P-384 + SHA-384 based logic.
  - Using Rust's `p384` and `sha2` crates, or the Cloudflare `voprf` crate.
  - Implementing the three mathematical steps: `Blind()` (client), `Evaluate()` (server), and `Unblind()` (client).
  - **Memory Safety (`zeroize`):** Actively wiping sensitive variables from RAM immediately after execution. Specific targets: the `blind_factor` during tokenization, the raw `oprf_key` in the server, and the final `plaintext_score` decrypted by the KMS.
- **My Deliverable:** The `crates/crypto` shared library.

### Phase 2: OPRF Client & API Contracts (Axum & Serde)
- **Goal:** Build the interface that the `ingestion-service` will use to interact with OPRF.
- **Key Topics:**
  - `serde` and JSON Schema validation for defining exact payload shapes.
  - Writing `axum` REST handlers to receive blinded inputs and return evaluated outputs.
  - Designing API contracts in `schemas/openapi/` and `schemas/json-schema/`.
- **My Deliverables:** 
  - The `crates/oprf-client` shared library (providing clean wrapper functions for downstream teams).
  - The `services/oprf-service` (Axum REST API handling evaluations separated from the main API server).

### Phase 3: The External KMS Bridge (gRPC, Tonic & Prost)
- **Boundary Clarification:** Zama KMS *does* all the heavy TFHE math (LWE, noise flooding, Lagrange interpolation). My job is merely to build the Orchestrator Client.
- **Goal:** Build the client that communicates with the external Zama Threshold KMS cluster. 
- **Key Topics:**
  - *Note: `prism` uses REST internally, but the external Zama KMS speaks gRPC.*
  - Extracting `.proto` files (e.g., `kms-service.v1.proto`) from `00_Reference/repos/kms/core/grpc/proto/`.
  - Compiling `.proto` files into native Rust using `tonic-prost-build` in a `build.rs` script.
  - **Client Orchestration:** Studying `core-client/src/decrypt.rs` from the Zama KMS repo. I must fan out requests to 4 KMS nodes, wait for `t` responses, and aggregate the threshold shares.
  - **EIP-712 Signatures:** Using `alloy-sol-types` and `alloy-signer` to correctly format KMS requests and verify the `external_signature` on KMS responses.
- **My Deliverable:** The `crates/kms-client` shared library.

### Phase 4: Event-Driven Integration (Redpanda / Kafka)
- **Goal:** Connect the KMS client into the asynchronous workflows of `prism`.
- **Key Topics:**
  - Integrating `tracing`, `opentelemetry`, and structured logging.
  - Formatting detailed application errors with `thiserror` and `anyhow`.
  - Consuming `kms.decrypt.requested.v1` Kafka events.
  - Triggering the threshold decryption via `kms-client` and emitting `kms.decrypt.completed.v1`.
- **My Deliverable:** Event listeners and handlers wiring `kms-client` into the larger system.

### Phase 5: Local Integration & Sandboxing (Docker Compose)
- **Goal:** Ensure the cryptographic boundaries work with the full infrastructure.
- **Key Topics:**
  - Writing `#[tokio::test]` functions using `mockall` to simulate KMS cluster responses, ensuring downstream teams (like `disclosure`) can test their code without a real KMS.
  - Spinning up PostgreSQL, Redpanda, and MinIO via Docker Compose to test full event loop flows.