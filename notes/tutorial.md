# KMS Learning Plan

> **For agents starting a new chat**: read [agent-instructions.md](./agent-instructions.md) first.  
> A step-by-step tree guide. Confirm each layer before going deeper.  
> **Current depth → Layer 4 (KeyGen + Encrypt/Decrypt deep dives).**
>
> In a hurry? [kms_crypto_notes.md](./kms_crypto_notes.md) is a condensed, one-page version of everything below Layer 3 — read that instead if you just want the crypto, not the click-through commands.


---

## The Big Picture (What We Are Learning)

```
KMS System
├── Mode A: Centralized (1 server)
└── Mode B: Threshold (4 servers, MPC)

Each mode has 3 phases:
├── Phase 1: KeyGen      — server generates FHE key pair
├── Phase 2: Encryption  — client encrypts data using public key
└── Phase 3: Decryption  — server decrypts ciphertext using secret key
```

For both modes, the pipeline is always:
```
Compile → Run Server → Run Client
```

---

## Layer 0 — The Concept Map (Start Here)

Before touching any code, understand the 5 actors:

| Actor                | What It Is                                                      | Lives In                           |
| -------------------- | --------------------------------------------------------------- | ---------------------------------- |
| `kms-gen-keys`       | One-shot tool: creates server signing keys                      | `core/service` binary              |
| `kms-server`         | The always-running KMS server                                   | `core/service` binary              |
| `kms-core-client`    | The CLI tool you use to send commands                           | `core-client` binary               |
| **Centralized mode** | A single `kms-server` that holds the full secret key            | Config: `default_centralized.toml` |
| **Threshold mode**   | 4 `kms-server` nodes that each hold a *share* of the secret key | Config: `compose_1..4.toml`        |

> **Key insight**: In threshold mode, no single server ever has the full secret key. You need ≥3 nodes cooperating to decrypt anything. That is the security guarantee.

✅ **Confirm before moving on**: Can you explain in one sentence why threshold mode is safer than centralized?
Ans: only if (t + 1) nodes are compromised will the private key be reconstructed, the private key isn't storing in 1 place 

---

## Layer 1 — Compile

> Goal: install dependencies, then build all three binaries from source.

### Step 0 — Install System Dependencies

These must be present before `cargo build` will succeed:

```bash
# Ubuntu / GitHub Codespace
sudo apt-get update && sudo apt-get install -y \
  protobuf-compiler \   # protoc: needed to compile .proto gRPC definitions
  pkg-config \          # lets Rust find system libraries
  libssl-dev \          # OpenSSL headers (for TLS)
  build-essential       # gcc, make, etc.

# macOS (Homebrew)
brew install protobuf pkg-config openssl
```

Also install Rust itself if not already present:
```bash
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh -s -- -y
source "$HOME/.cargo/env"
```

> The Rust **version** is pinned in `rust-toolchain.toml` at the repo root.  
> `rustup` picks it up automatically — you don't need to set it manually.

Optionally, add swap space to prevent out-of-memory crashes during compilation (recommended on Codespace):
```bash
sudo fallocate -l 8G /tmp/swapfile && sudo chmod 600 /tmp/swapfile
sudo mkswap /tmp/swapfile && sudo swapon /tmp/swapfile
```

---

### Step 1 — Compile

```bash
cargo build --bin kms-gen-keys --bin kms-server --bin kms-core-client
```

- **`cargo build`** — downloads all Rust crate dependencies, then compiles.
- **`--bin X`** — only build binary `X` (skips library-only crates, faster).
- First build: **15–30 minutes**. Subsequent builds: seconds (only changed files recompile).

### What gets produced

```
target/debug/
├── kms-gen-keys      ← one-shot signing key generator (run before first server start)
├── kms-server        ← the long-running server binary
└── kms-core-client   ← the CLI tool you send commands with
```

> **About `target/` and `cargo run`:**  
> `target/debug/kms-server` is a real native binary — you can run it directly:  
> `./target/debug/kms-server --config-file ...`  
> `cargo run --bin kms-server -- --config-file ...` does exactly the same thing: compile (if needed) then run.  
> In production Docker builds, the binary is copied out of `target/` and executed directly — no Cargo involved.

> Completed in GitHub Codespace. Binaries are large (~hundreds of MB) due to FHE crypto crates. ✅

---


## Layer 2 — Run Server

> All commands run from the **repo root** (`/workspaces/kms` on Codespace).

### 2A. Centralized (Single Node) — Start Here

Two steps before clients can connect:

**Step 1 — Generate the server's signing key (run once):**
```bash
cargo run --bin kms-gen-keys -- centralized
```
- Writes `./keys/PUB/` (verification key) and `./keys/PRIV/` (signing key).
- The server uses this ECDSA key to sign every FHE key it generates (EIP-712).
- If `./keys/` already has material, this is a no-op (safe to re-run).

**Step 2 — Start the server (keep this terminal open):**
```bash
cargo run --bin kms-server -- --config-file core/service/config/default_centralized.toml
```
- Listens on `0.0.0.0:50051` (gRPC).
- Ready when you see: `Starting centralized KMS server v...`

---

### 2B. Threshold (4 Nodes)

**Step 1 — Generate per-party signing keys + mTLS certs:**
```bash
for i in 1 2 3 4; do
  cargo run --bin kms-gen-keys -- \
    --private-storage file --private-file-path ./keys \
    --public-storage  file --public-file-path  ./keys \
    threshold --signing-key-party-id $i --tls-subject dev-kms-core-$i.com
done
```

**Step 2 — Add hostname aliases (needed for mTLS cert matching):**
```bash
echo "127.0.0.1 abcd.dev-kms-core-1.com abcd.dev-kms-core-2.com abcd.dev-kms-core-3.com abcd.dev-kms-core-4.com" \
  | sudo tee -a /etc/hosts
```

**Step 3 — Start all 4 nodes (one per terminal):**
```bash
# Terminal 1
cargo run --bin kms-server -- --config-file core/service/config/compose_1.toml

# Terminal 2
cargo run --bin kms-server -- --config-file core/service/config/compose_2.toml

# Terminal 3
cargo run --bin kms-server -- --config-file core/service/config/compose_3.toml

# Terminal 4
cargo run --bin kms-server -- --config-file core/service/config/compose_4.toml
```
- Node 1 listens on gRPC port `50100`, MPC port `50001`
- Node 2: gRPC `50200`, MPC `50002`
- Node 3: gRPC `50300`, MPC `50003`
- Node 4: gRPC `50400`, MPC `50004`
- All 4 nodes are ready when each prints: `Starting threshold KMS server v...`

✅ **Confirm before moving on**: Pick one mode. Get the server(s) running. Then move to Layer 3.

---

## Layer 3 — Run Client (3 Phases)

> Open a **new terminal** tab. All commands below are for **centralized mode**.  
> For threshold, replace `-f core-client/config/client_local_centralized.toml`  
> with `-f core-client/config/client_local_threshold.toml`.

### Phase 1: KeyGen

Key generation is a two-step handshake:

```
Step 1: Preprocessing  → returns PREPROC_ID
Step 2: KeyGen         → consumes PREPROC_ID, returns KEY_ID
```

---

#### 1. The Core Commands (How to run)

**Step 1 — Preprocessing:**
```bash
cargo run --bin kms-core-client -- \
  -f core-client/config/client_local_centralized.toml \
  insecure-preproc-key-gen
```
Output: `insecure preproc done - "request_id": "<PREPROC_ID>"`

**Step 2 — Key Generation:**
```bash
cargo run --bin kms-core-client -- \
  -f core-client/config/client_local_centralized.toml \
  insecure-key-gen --preproc-id <PREPROC_ID>
```
Output: `insecure keygen done - "request_id": "<KEY_ID>"`

---

#### 2. The Crypto Behind It (short version)

The LWE key generation equation is $\mathbf{b} = \mathbf{A}\mathbf{s} + \mathbf{e} \pmod q$. 

**Centralized**: one server samples $\mathbf{s}, \mathbf{e}$ directly in the KeyGen step and computes everything itself. 

**Threshold**: $\mathbf{s}, \mathbf{e}$ are generated as Shamir secret shares across nodes during Preprocessing; KeyGen then builds the LWE public key and key-switching key locally per node (both linear, no MPC), and the Bootstrap Key via a real MPC step (Beaver-triple multiply) — the only non-linear part of the whole flow. Secret key coefficients are **binary** ($s_i \in \{0,1\}$), which turns key-multiplication into a skip/add and keeps noise growth bounded.

> 🔍 **Full detail** (math-to-code mapping, call traces, Mermaid diagrams, Beaver-triple protocol, why coefficients are binary not ternary): [keygen_deep_dive.md](./keygen_deep_dive.md), [threshold_dkg_deep_dive.md](./threshold_dkg_deep_dive.md), or the condensed [kms_crypto_notes.md](./kms_crypto_notes.md#1-the-math-lwe-learning-with-errors).

---

### Phase 2: Encryption

> **The server is NOT contacted during encryption.**  
> The client reads the public key from `./keys/PUB/` and encrypts entirely locally.

```bash
cargo run --bin kms-core-client -- \
  -f core-client/config/client_local_centralized.toml \
  encrypt \
  --to-encrypt 0500000000000000 \
  --data-type euint64 \
  --key-id <KEY_ID> \
  --ciphertext-output-path ./ciphertext.bin
```

- `0500000000000000` = the number `5` as 8-byte little-endian hex
- `--data-type euint64` = encrypted unsigned 64-bit integer
- Output: a binary blob written to `./ciphertext.bin`

> **Why is this client-only?** Public key cryptography: anyone with the public key can encrypt. Only the server (holding the secret key `s`) can decrypt.

---

> 🔍 **Deep Dive**: for the encryption code path and both threshold decryption protocols (public-decrypt vs. user-decrypt), read [encrypt_decrypt_deep_dive.md](./encrypt_decrypt_deep_dive.md).

---

### Phase 3: Decryption

> **The server IS contacted here.** It uses the secret key to decrypt.

```bash
cargo run --bin kms-core-client -- \
  -f core-client/config/client_local_centralized.toml \
  public-decrypt from-file \
  --input-path ./ciphertext.bin
```

Expected output:
```
plaintexts: [TypedPlaintext { bytes: [5, 0, 0, 0, 0, 0, 0, 0], fhe_type: 5 }]
```
The `bytes: [5, 0, 0, 0, 0, 0, 0, 0]` is `5` in little-endian — matching what we encrypted. ✅

---

### Full Sequence (Copy-Paste Ready)

```bash
# ── SERVER (Terminal 1) ───────────────────────────────────────────
cargo run --bin kms-gen-keys -- centralized
cargo run --bin kms-server -- --config-file core/service/config/default_centralized.toml

# ── CLIENT (Terminal 2) ───────────────────────────────────────────

# Phase 1a: Preprocessing
cargo run --bin kms-core-client -- \
  -f core-client/config/client_local_centralized.toml \
  insecure-preproc-key-gen
# → Copy PREPROC_ID from output

# Phase 1b: KeyGen
cargo run --bin kms-core-client -- \
  -f core-client/config/client_local_centralized.toml \
  insecure-key-gen --preproc-id <PREPROC_ID>
# → Copy KEY_ID from output

# Phase 2: Encrypt
cargo run --bin kms-core-client -- \
  -f core-client/config/client_local_centralized.toml \
  encrypt \
  --to-encrypt 0500000000000000 \
  --data-type euint64 \
  --key-id <KEY_ID> \
  --ciphertext-output-path ./ciphertext.bin

# Phase 3: Decrypt
cargo run --bin kms-core-client -- \
  -f core-client/config/client_local_centralized.toml \
  public-decrypt from-file \
  --input-path ./ciphertext.bin
```

✅ **Confirm before moving on**: Run all 3 phases. Encrypt `5`, decrypt it, confirm you get `5` back.
Ans: skip actual command running

---

## Layer 4 — Go Deeper (Choose Your Path)

Once you complete Layers 0–3, pick what to explore next:

```
Option A: Code Walkthrough — Centralized & Threshold KeyGen
  → Read the [KeyGen Deep Dive](./keygen_deep_dive.md) (math-to-code mapping & call traces)
  → Read raw/kms-keygen-simple.md (beginner-friendly story)
  → Read raw/kms-keygen-deep-dive.md (detailed centralized function-by-function trace)

Option B: Threshold MPC — How 4 nodes agree on a key
  → Read threshold_dkg_deep_dive.md (corrected 2026-07-13; raw/01_keygen_threshold.md is
     superseded/inaccurate — it describes the wrong scheme, kept only for archive)
  → Read the Noah's Ark paper (the MPC protocol behind DKG)

Option C: The Math — LWE, public keys, noise
  → Read raw/lwe-math-to-code-mapping.md (has a couple of minor citation errors — see the
     erratum note at the top of keygen_deep_dive.md; the core b=As+e math there is fine)
  → Understand why b = As + e is computationally hard to invert

Option D: Encrypt/Decrypt — client-side encryption, threshold noise-flooding decrypt
  → Read encrypt_decrypt_deep_dive.md (encryption code path + public-decrypt vs.
     user-decrypt threshold protocols)

Option E: Just want the condensed version of all of the above?
  → Read kms_crypto_notes.md — one page, no click-through, covers §1-6 of every deep dive
```

---

## Quick Reference — Commands Cheat Sheet

| Task | Command |
|---|---|
| Build all binaries | `cargo build --bin kms-gen-keys --bin kms-server --bin kms-core-client` |
| Gen signing keys (central) | `cargo run --bin kms-gen-keys -- centralized` |
| Start central server | `cargo run --bin kms-server -- --config-file core/service/config/default_centralized.toml` |
| Preprocessing | `cargo run --bin kms-core-client -- -f core-client/config/client_local_centralized.toml insecure-preproc-key-gen` |
| Key generation | `cargo run --bin kms-core-client -- -f core-client/config/client_local_centralized.toml insecure-key-gen --preproc-id <PREPROC_ID>` |
| Encrypt | `cargo run --bin kms-core-client -- -f core-client/config/client_local_centralized.toml encrypt --to-encrypt 0500000000000000 --data-type euint64 --key-id <KEY_ID> --ciphertext-output-path ./ct.bin` |
| Decrypt | `cargo run --bin kms-core-client -- -f core-client/config/client_local_centralized.toml public-decrypt from-file --input-path ./ct.bin` |
