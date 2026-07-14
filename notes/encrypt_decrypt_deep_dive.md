# Encrypt / Decrypt Deep Dive

> Client-side encryption and both threshold decryption protocols, with code citations.
>
> **Back to:** [tutorial.md](./tutorial.md)

---

## 1. Encryption — Same Code Path for Both Modes

Encryption **never contacts a server**. The client only needs the public key.

```
client downloads public key (S3/HTTP fetch, not gRPC)
  → tfhe-rs CompactPublicKey.encrypt(message).expand()
```

This uses tfhe-rs's *compact public-key encryption* format (packs many ciphertexts against a shared seed, then expands them). Internally this is still LWE-style encryption of the message under $(\mathbf{A}, \mathbf{b})$ — but that exact arithmetic lives inside the `tfhe-rs` library dependency, not this repo.

Because the combined public key is one artifact assembled once at KeyGen, centralized and threshold clients encrypt with **the exact same code path** — the only difference is which node's storage bucket the key is fetched from.

Code: `core-client/src/lib.rs` (`encrypt`), `core/threshold-execution/src/tfhe_internals/utils.rs` (`expanded_encrypt`).

Command (see [tutorial.md § Phase 2](./tutorial.md) for the full walkthrough):
```bash
cargo run --bin kms-core-client -- \
  -f core-client/config/client_local_centralized.toml \
  encrypt --to-encrypt 0500000000000000 --data-type euint64 \
  --key-id <KEY_ID> --ciphertext-output-path ./ciphertext.bin
```

---

## 2. Decryption — Centralized: Trivial, Local

One server, full key → `hl_ct.decrypt(&client_key)` (plain tfhe-rs API call). No networking, no MPC.

Code: `core/service/src/engine/centralized/central_kms.rs`, entrypoint `core/service/src/engine/centralized/service/decryption.rs`.

---

## 3. Decryption — Threshold Mode: Noise-Flooding MPC

No node has the full key, so decrypting is a protocol, not a function call. Every node computes a **partial decryption** using only its own share, then masks it with a big pre-shared random value (hides what that one share would otherwise leak). Then it branches into two modes:

| | **public-decrypt** | **user-decrypt** |
|---|---|---|
| Who ends up seeing the plaintext | every node (and thus whoever asks) | only the requesting client |
| Extra MPC step | nodes run an "open" (`robust_open_list_to_all`) so all nodes learn the same plaintext, each signs it | **no open step** — each node just signcrypts its own raw masked share under the client's ephemeral public key |
| Client-side work | none — take one signed response | collect ≥ t+1 signcrypted shares, decrypt each with its ephemeral private key, then **Shamir-reconstruct** the plaintext itself |
| Use case | results meant to be public (e.g. on-chain reveal) | private results — this is the modern name for what used to be called "reencryption" |

Code: `core/threshold-execution/src/endpoints/decryption_non_wasm.rs` (`partial_decrypt128/64`, `DecryptionMode::NoiseFloodSmall/Large`); handlers in `core/service/src/engine/threshold/service/{public_decryptor,user_decryptor}.rs`; client-side reconstruction in `core/service/src/client/user_decryption_wasm.rs`.

---

## 4. Key Files to Explore

1. **Encrypt entrypoint**: `core-client/src/lib.rs` (`encrypt`)
2. **Centralized decrypt**: `core/service/src/engine/centralized/service/decryption.rs`, `core/service/src/engine/centralized/central_kms.rs`
3. **Threshold decrypt protocol**: `core/threshold-execution/src/endpoints/decryption_non_wasm.rs`
4. **Public-decrypt handler**: `core/service/src/engine/threshold/service/public_decryptor.rs`
5. **User-decrypt handler**: `core/service/src/engine/threshold/service/user_decryptor.rs`
6. **Client-side Shamir reconstruction**: `core/service/src/client/user_decryption_wasm.rs`
