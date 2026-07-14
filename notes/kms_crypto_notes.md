# KMS Threshold Cryptography — Math & Code, in One Note

> One-stop, condensed version of everything in `notes/` (tutorial.md, keygen_deep_dive.md, threshold_dkg_deep_dive.md, encrypt_decrypt_deep_dive.md). Read this first; follow the links only when you want the full code trace on one topic.

## 0. The Shape of the System

```
KMS
├── Centralized mode  — 1 server holds the whole secret key
└── Threshold mode    — N servers (e.g. 4), each holds only a SHARE; ≥ t+1 must cooperate

Every mode, 3 operations:
1. KeyGen    — produce a public key (to encrypt) + secret key/shares (to decrypt)
2. Encrypt   — client-only, using the public key
3. Decrypt   — server(s) use the secret key/shares to recover the plaintext
```

**Security idea of threshold mode**: the secret key never exists whole, anywhere, ever. It's split into shares at generation time (DKG) and every operation that needs the secret key is done as a multi-party computation (MPC) over the shares. An attacker must compromise more than a threshold number of servers to learn anything.

---

## 1. The Math: LWE (Learning With Errors)

FHE keys here are built on the **LWE hardness assumption**:

$$ \mathbf{b} = \mathbf{A}\mathbf{s} + \mathbf{e} \pmod q $$

| Symbol | Meaning | Public or secret? |
|---|---|---|
| $\mathbf{s}$ | secret key, a vector of **binary** coefficients $s_i \in \{0,1\}$ | secret |
| $\mathbf{A}$ | random matrix (the "mask") | public |
| $\mathbf{e}$ | small random noise | secret (but small enough to not matter once added) |
| $\mathbf{b}$ | $= \mathbf{A}\mathbf{s}+\mathbf{e}$ — looks random without $\mathbf{s}$ | public |

$(\mathbf{A}, \mathbf{b})$ together **are** the public key. Recovering $\mathbf{s}$ from $(\mathbf{A}, \mathbf{b})$ is the LWE problem — believed hard even for quantum computers.

**Why binary $s_i$?** Two payoffs used throughout the codebase:
- No multiplication needed when applying the key: $s_i=0$ → skip, $s_i=1$ → add. Much cheaper than general polynomial multiplication.
- Keeps noise growth small and predictable (needed so decryption doesn't fail).

Besides the plain LWE public key, TFHE (the scheme this KMS uses) needs two more per-key artifacts, generated once at KeyGen time and reused for every future homomorphic operation:

| Artifact | Purpose |
|---|---|
| **Bootstrap Key (BK)** | GGSW-encrypts each bit of the LWE key under a *second* key (the GLWE key) — enables "programmable bootstrapping" (refreshing noise after homomorphic ops) |
| **Key-Switching Key (KSK)** | Linearly re-encrypts data from one key's "coordinates" to another's |

---

## 2. Centralized vs. Threshold — Where Each Piece Is Computed

| Step | Centralized | Threshold (DKG) |
|---|---|---|
| $\mathbf{s}$, $\mathbf{e}$ | sampled directly by the one server, in KeyGen | generated as **Shamir secret shares** across N nodes, in a **Preprocessing** phase, *before* KeyGen |
| $\mathbf{A}$ | constructed on the fly | constructed on the fly (public, so no sharing needed) |
| $\mathbf{b} = \mathbf{A}\mathbf{s}+\mathbf{e}$ | computed directly | computed **locally per node** from its own share — linear, so no MPC needed |
| Key-Switching Key | computed directly | computed **locally per node** — also linear |
| Bootstrap Key | computed directly | needs a **real MPC step** (see §3) — the only non-linear part of KeyGen |
| Decrypt | direct local decrypt | MPC "noise-flooding" protocol (see §5) |

Centralized "preprocessing" is a no-op placeholder that just books a request ID; threshold preprocessing does the real cryptographic heavy-lifting up front so the online KeyGen step is fast.

---

## 3. Threshold Key Generation (DKG)

**Shamir secret sharing** (the sharing scheme used for every secret value: key bits, noise, MPC helper values): a secret $x$ is the constant term of a random degree-$t$ polynomial $P$; node $j$ holds $P(j)$. Any $t{+}1$ shares reconstruct $x = P(0)$ via Lagrange interpolation. Nodes can add shares or multiply by public constants **locally, for free** — but multiplying two *secret* values requires help (next paragraph).

**Beaver triples** — the trick for multiplying two secret-shared values without revealing them. Precompute a random shared triple $(x, y, z)$ with $z = xy$. To multiply secret shares $k_1, k_2$: jointly reveal $\varepsilon = k_1 - x$ and $\rho = k_2 - y$ (these reveal nothing about $k_1,k_2$ since $x,y$ are random and secret), then each node computes its share of $p = k_1 k_2$ locally: $\text{share}_j(p) = z_j + k_{2,j}\varepsilon - x_j \rho$.

**Why KeyGen needs this**: building the Bootstrap Key means GGSW-encrypting (GLWE-key bit) × (LWE-key bit) — a product of two *secret-shared* values. That's the **only** multiplication in the whole KeyGen flow; the public key and key-switching key are purely linear and need no MPC.

```
Preprocessing (offline, per DKG session):
  nodes jointly generate → binary bit-shares of s (LWE + GLWE) · noise shares · Beaver triples
  → stored, tagged with a PREPROC_ID

KeyGen (online, consumes PREPROC_ID):
  1. LWE public key  b = As + e     — local per node, no MPC
  2. Key-switching key              — local per node, no MPC
  3. Bootstrap key (GGSW)            — MPC: Beaver-triple multiply, only non-linear step
  4. each node writes its share of s to its own private vault — no node ever has the whole key
```

Code: `core/threshold-execution/src/endpoints/keygen.rs` (`generate_private_key_set`, `generate_all_compressed_public_keys`); triples in `online/triple.rs`; GGSW multiply in `tfhe_internals/ggsw_ciphertext.rs`. Full trace: [threshold_dkg_deep_dive.md](./threshold_dkg_deep_dive.md), [keygen_deep_dive.md](./keygen_deep_dive.md).

---

## 4. Encryption (identical in both modes)

Encryption **never contacts a server**. The client just needs the public key.

```
client downloads public key (S3/HTTP fetch, not gRPC)
  → tfhe-rs CompactPublicKey.encrypt(message).expand()
```

This uses tfhe-rs's *compact public-key encryption* format (packs many ciphertexts against a shared seed, then expands them). Internally this is still LWE-style encryption of the message under $(\mathbf{A}, \mathbf{b})$ — but that exact arithmetic lives inside the `tfhe-rs` library dependency, not this repo.

Because the combined public key is one artifact assembled once at KeyGen, centralized and threshold clients encrypt with **the exact same code path** — the only difference is which node's storage bucket the key is fetched from.

Code: `core-client/src/lib.rs` (`encrypt`), `core/threshold-execution/src/tfhe_internals/utils.rs` (`expanded_encrypt`). Full trace: [encrypt_decrypt_deep_dive.md](./encrypt_decrypt_deep_dive.md#1-encryption--same-code-path-for-both-modes).

---

## 5. Decryption

### Centralized: trivial, local
One server, full key → `hl_ct.decrypt(&client_key)` (plain tfhe-rs API call). No networking, no MPC.

### Threshold: noise-flooding MPC
No node has the full key, so decrypting is a protocol, not a function call. Every node computes a **partial decryption** using only its own share, then masks it with a big pre-shared random value (hides what that one share would otherwise leak). Then it branches into two modes:

| | **public-decrypt** | **user-decrypt** |
|---|---|---|
| Who ends up seeing the plaintext | every node (and thus whoever asks) | only the requesting client |
| Extra MPC step | nodes run an "open" (`robust_open_list_to_all`) so all nodes learn the same plaintext, each signs it | **no open step** — each node just signcrypts its own raw masked share under the client's ephemeral public key |
| Client-side work | none — take one signed response | collect ≥ t+1 signcrypted shares, decrypt each with its ephemeral private key, then **Shamir-reconstruct** the plaintext itself |
| Use case | results meant to be public (e.g. on-chain reveal) | private results — this is the modern name for what used to be called "reencryption" |

Code: `core/threshold-execution/src/endpoints/decryption_non_wasm.rs` (`partial_decrypt128/64`, `DecryptionMode::NoiseFloodSmall/Large`); handlers in `core/service/src/engine/threshold/service/{public_decryptor,user_decryptor}.rs`; client-side reconstruction in `core/service/src/client/user_decryption_wasm.rs`. Full trace: [encrypt_decrypt_deep_dive.md](./encrypt_decrypt_deep_dive.md#3-decryption--threshold-mode-noise-flooding-mpc).

---

## 6. Code Map (where to actually look)

```
core/
├── service/src/engine/
│   ├── centralized/                 ← 1-server mode
│   │   ├── service/key_gen.rs         entrypoint: KeyGen
│   │   ├── service/decryption.rs      entrypoint: public/user decrypt
│   │   └── central_kms.rs             actual crypto calls (ClientKey/FhePublicKey/decrypt)
│   └── threshold/                   ← N-server MPC mode
│       └── service/
│           ├── preprocessor.rs        DKG preprocessing entrypoint
│           ├── key_generator.rs       DKG keygen entrypoint
│           ├── public_decryptor.rs    public-decrypt entrypoint
│           └── user_decryptor.rs      user-decrypt entrypoint
├── threshold-execution/src/          ← the actual MPC math/protocols
│   ├── endpoints/keygen.rs            DKG key-share generation
│   ├── endpoints/decryption_non_wasm.rs  noise-flooding decrypt protocol
│   ├── online/triple.rs               Beaver triple protocol
│   └── tfhe_internals/                LWE/GLWE/GGSW key struct math
└── ...
core-client/src/lib.rs                ← CLI: keygen / encrypt / decrypt commands
```

---

## 7. Further Reading (full code traces, more diagrams)

- [tutorial.md](./tutorial.md) — step-by-step build/run/client walkthrough
- [keygen_deep_dive.md](./keygen_deep_dive.md) — KeyGen call traces + Mermaid diagrams
- [threshold_dkg_deep_dive.md](./threshold_dkg_deep_dive.md) — DKG math + code, line-cited
- [encrypt_decrypt_deep_dive.md](./encrypt_decrypt_deep_dive.md) — encrypt/decrypt call traces + threshold decryption protocols
