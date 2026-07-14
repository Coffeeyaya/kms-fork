# KMS Learning Log (fresh start)

> Live journal for this teaching session. Grows one small chunk at a time — I write a bit, you read it and react, then I add the next bit. Say "new file" whenever you want to close this one off and start another.
> Older notes (`tutorial.md`, `keygen_deep_dive.md`, etc.) are kept as backup/reference, not part of this flow.

---

## Chunk 1 — What are we even building?

Before installing anything: the KMS repo produces **3 separate programs** (Rust calls them "binaries"). You'll compile all 3 today.

| Binary            | Job                                                                                                                               |
| ----------------- | --------------------------------------------------------------------------------------------------------------------------------- |
| `kms-gen-keys`    | Run once. Creates the server's *signing* key (not the encryption key — just proves "this response really came from this server"). |
| `kms-server`      | The long-running program. Waits for requests and does the actual crypto (KeyGen / Encrypt / Decrypt).                             |
| `kms-core-client` | The CLI you type commands into, to talk to `kms-server`.                                                                          |

So the order of operations for the whole project is always:
```
compile → run kms-server → use kms-core-client to send it commands
```

That's it for chunk 1 — just the mental model, no commands yet.

**Check-in:** does it make sense why signing keys (`kms-gen-keys`) are a separate thing from the encryption keys we'll generate later? Yes.

---

## Chunk 2 — Installing dependencies

All 3 binaries are Rust programs, so first you need **Rust itself** (the compiler toolchain):

```bash
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh -s -- -y
source "$HOME/.cargo/env"
```

>[!note]
>**The config file contains a command:** `export PATH="$HOME/.cargo/bin:$PATH"`.
**The `source` command runs that script inside your active terminal** (instead of a background sub-process).
**Result:** Your current terminal's `PATH` variable is updated with the folder where Rust commands live, making `cargo` instantly usable.

Besides Rust, the build needs a few **system-level** tools already installed on your machine (Rust's package manager, `cargo`, can't install these — they're OS packages):

| Tool                           | Why the build needs it                                                                                                   |
| ------------------------------ | ------------------------------------------------------------------------------------------------------------------------ |
| `protobuf-compiler` (`protoc`) | The KMS server talks gRPC — its API is defined in `.proto` files, and `protoc` turns those into Rust code at build time. |
| `pkg-config`                   | Helps Rust's build scripts *find* other system libraries (like OpenSSL) on your machine.                                 |
| `libssl-dev` (OpenSSL headers) | Needed for TLS — the server talks to clients/other nodes over encrypted connections.                                     |
| `build-essential`              | Generic C compiler + linker (`gcc`, `make`) — some Rust crates wrap C code and need these to build.                      |

Install command:
```bash
# Ubuntu / GitHub Codespace
sudo apt-get update && sudo apt-get install -y protobuf-compiler pkg-config libssl-dev build-essential

# macOS (Homebrew)
brew install protobuf pkg-config openssl
```

**Check-in:** got these installed OK, or did any of them fail / already exist on your machine?
Since I don't have a server now, so you can just explain, and I learn, but skip actual code running.

---

## Chunk 3 — Compiling

One command builds all 3 binaries:

```bash
cargo build --bin kms-gen-keys --bin kms-server --bin kms-core-client
```

- `cargo` is Rust's build tool — it reads `Cargo.toml`, downloads every dependency crate the project needs, then compiles everything.
- `--bin X` says "only build binary X". The repo actually contains more buildable things than just these 3 (libraries, other tools), so this flag skips the ones you don't need — faster than building everything.
- First time: **15-30 minutes** (compiling FHE crypto crates is heavy). After that, `cargo` only recompiles files you've changed, so it's seconds.

Where the output goes:
```
target/debug/
├── kms-gen-keys
├── kms-server
└── kms-core-client
```
These are real, standalone native programs — you could copy `target/debug/kms-server` anywhere and run it directly, no `cargo` needed at that point. `cargo run --bin X -- <args>` is just a shortcut for "rebuild X if needed, then execute it with these args."

**Check-in:** clear on what `cargo build` actually produces, and why `target/debug/` has 3 files matching the 3 binaries from Chunk 1? very clear

---

## Chunk 4 — Running the server (centralized mode)

There are two modes the server can run in — **centralized** (1 server, simplest) or **threshold** (4 servers doing MPC). Start with centralized; threshold is its own chunk later.

Two steps, run once each:

**Step 1 — generate the server's signing key:**
```bash
cargo run --bin kms-gen-keys -- centralized
```
This is the `kms-gen-keys` binary from Chunk 1. It creates an ECDSA key pair and writes it to `./keys/PUB/` (public, shareable) and `./keys/PRIV/` (private, stays on this machine). Every FHE key the server generates later gets signed with this, so a client can verify "this key really came from this server" (the signing scheme is called EIP-712 — same one Ethereum wallets use to sign structured messages).

**Step 2 — start the server:**
```bash
cargo run --bin kms-server -- --config-file core/service/config/default_centralized.toml
```
This starts `kms-server` and makes it listen on port `50051` using gRPC (a request/response protocol, like a more structured HTTP). It reads `default_centralized.toml` to know things like which port to use and where to store keys. You'll know it's ready when it prints `Starting centralized KMS server v...`, and this terminal stays occupied — the server just sits there waiting for requests.

**Check-in:** does the signing-key step (Step 1) make sense as a one-time setup, separate from Step 2 which runs forever? yes

---

## Chunk 5 — Running the server (threshold mode, 4 nodes)

Threshold mode is the same idea as Chunk 4, just done **4 times**, once per node — because no single node is allowed to hold the whole secret key.

**Step 1 — generate a signing key + mTLS cert for each of the 4 parties:**
```bash
for i in 1 2 3 4; do
  cargo run --bin kms-gen-keys -- \
    --private-storage file --private-file-path ./keys \
    --public-storage  file --public-file-path  ./keys \
    threshold --signing-key-party-id $i --tls-subject dev-kms-core-$i.com
done
```
Same idea as centralized Step 1 (signing keys), but now each node gets its own identity (`party-id $i`), plus a TLS certificate. The cert matters here because in threshold mode the nodes don't just talk to a client — they talk **to each other** (that's the MPC part), and those node-to-node connections use **mTLS**: both sides present a certificate, so node 2 can prove to node 1 "I really am node 2," not just any random party.

> [!note]
> Q: --private-storage, --public-storage and threshold, are they terminal keyword, or just arguments pass into kms-gen-keys? and what's --tls-subject?
> A: Looked at the actual CLI definition (`core/service/src/bin/kms-gen-keys.rs`) to be precise:
 > * `centralized` and `threshold` **are subcommands** — literal keywords the CLI parser recognizes (Rust's `clap` library, `#[clap(subcommand)] mode: Mode`). You must pick exactly one; they're not just arbitrary text.
> * `--private-storage` / `--public-storage` are **top-level flags**, valid for either subcommand. They choose *where* the keys get written: `file` (local disk, the default) or `s3` (a cloud bucket). `--private-file-path` / `--public-file-path` are the actual directory paths, only used when storage is `file`.
> * `--tls-subject` and `--signing-key-party-id` are flags **specific to the `threshold` subcommand** (they don't exist under `centralized`, since only threshold mode needs per-party TLS certs). 
> * `--signing-key-party-id` is just this party's number (1-4). 
> * `--tls-subject` is the identity string baked into the self-signed TLS certificate this command generates for that party — later, when node 2 connects to node 1, node 1's TLS handshake checks "does the cert's subject match the hostname I expected for node 2?" That's why it's set to `dev-kms-core-$i.com` — matching the hostnames used in Step 2/3.


**Step 2 — make those TLS hostnames resolve locally:**
```bash
echo "127.0.0.1 abcd.dev-kms-core-1.com abcd.dev-kms-core-2.com abcd.dev-kms-core-3.com abcd.dev-kms-core-4.com" \
  | sudo tee -a /etc/hosts
```
* It routes the custom domain names used by your four KMS nodes back to your local machine (`127.0.0.1`).
Purely a local-dev workaround: the certs from Step 1 are issued for hostnames like `dev-kms-core-1.com`, but all 4 nodes are actually running on the same machine (`127.0.0.1`). This line tells your machine "when you see that hostname, it's really localhost" so the TLS handshake's hostname check passes.

> [! note]
Q: what's the purpose of this step? in reality, the nodes would run on multiple machines. How to change to that case?
> A: Each node's config (e.g. `core/service/config/compose_1.toml`) lists its 3 peers under `[[threshold.peers]]`, each with an `address` like `abcd.dev-kms-core-2.com` and a `port`. In this local tutorial, all 4 node *processes* are actually the same machine (`127.0.0.1`) just on different ports — but the peer list still refers to them by hostname (because that's also what the TLS cert's subject has to match, per the previous question). `/etc/hosts` is the trick that makes those made-up hostnames resolve to `127.0.0.1` instead of failing DNS lookup.
For a real multi-machine deployment, you'd change two things, and skip `/etc/hosts` entirely:
> 1. **Config**: each node's `[[threshold.peers]]` `address` becomes the peer's *real* reachable hostname or IP (e.g. an internal DNS name your infra already resolves, or a Kubernetes Service name) — not a fake `.com`.
> 2. **Certs**: `--tls-subject` at keygen time would be set to that same real hostname, and in production you'd likely use a real/internal CA to issue the certs rather than the demo self-signed one `kms-gen-keys` produces.
Real DNS (or your cluster's service discovery) then does the job, `/etc/hosts` is faking here — no manual hosts-file editing needed once hostnames are real.

**Step 3 — start all 4 nodes (one terminal each):**
```bash
cargo run --bin kms-server -- --config-file core/service/config/compose_1.toml   # node 1
cargo run --bin kms-server -- --config-file core/service/config/compose_2.toml   # node 2
cargo run --bin kms-server -- --config-file core/service/config/compose_3.toml   # node 3
cargo run --bin kms-server -- --config-file core/service/config/compose_4.toml   # node 4
```
Each node listens on **two** ports: one for gRPC (client-facing, e.g. node 1 = `50100`) and one for MPC (node-to-node, e.g. node 1 = `50001`). All 4 must be up before you can run any client command against threshold mode — since a request needs a threshold number of nodes cooperating to do anything.

> [!note]
> Q: you said client, is it the client terminal? and what's the threshold number in this case? if it's less than 4, not all nodes should present to decrypt?
>
A: Yes — "client" = `kms-core-client`, run from its own separate terminal, after all 4 server nodes are up. It's a different program from the 4 servers (that's Chunk 1's 3rd binary); it connects out to whichever node's gRPC port the client config points at.
> * For the threshold number: I checked `compose_1.toml` and it has `threshold = 1` under `[threshold]` — that's the actual parameter, with `n = 4` total parties. This `n = 3·t + 1` shape (4 = 3×1 + 1) is the standard bound for *robust* MPC protocols (the kind that tolerates actively malicious nodes, not just crashes): they're designed so that up to `t` parties can be corrupted or unresponsive, while the other `n − t` honest parties still complete the protocol correctly. So here, with `t = 1`, the protocol should tolerate **1** node being down and still finish with the remaining **3**.
>
> That's a bit different from "cryptographic reconstruction," which only strictly needs `t + 1 = 2` Shamir shares — but this protocol's guarantee is framed around honest-majority completion (`n − t`), not the bare reconstruction minimum, since it also has to detect/exclude a misbehaving party along the way. I haven't verified this by actually killing a node and testing — worth trying once you have the servers running, as a good hands-on check of what I just said.

> [!note]
> Q: Please correct me if I am wrong:
> Centralized case: 1 centralized server, 1 client. communication verified by ECDSA signing key.
> Threshold case: 4 servers (nodes), 1 client.
> * server (node) - client communication verified by ECDSA signing key.
> * server (node) - server (node) communication verified by TLS certificate.
> * when client sends a decryption query, which node will answer?
>
> A: Mostly right, two corrections after checking `core-client/src/decrypt.rs` and both client config files:
> 1. **"Verified by ECDSA" needs one refinement**: it's the *response payload* that's signed (EIP-712 signature over the specific KeyGen/Decrypt result), not the network channel itself. I grepped `core-client/src/*.rs` for any TLS client setup (`ClientTlsConfig`) and found none — the gRPC connection from client → node is **plain, unencrypted gRPC** in this local test setup (`address = "localhost:50100"`, no `https`). So client-node trust here = "I don't care if the channel is private, I just check the signature on what comes back really came from that server's key" — different guarantee than node-node's mTLS, which authenticates the *connection itself* before any data flows. Good catch to separate those two, they're not the same mechanism.
> 2. **Which node answers — none of them alone.** Checked `client_local_threshold.toml`: it lists **all 4** nodes' addresses under `[[cores]]`, and `core-client/src/decrypt.rs` spawns one request task per configured core (`JoinSet`, one `spawn` per `core_endpoints` entry) — so the client queries **all 4 nodes in parallel**, not just one. It then needs enough *valid, matching* responses back: the same config file sets `num_majority = 2` (min matching signed responses for `public-decrypt`) and `num_reconstruct = 3` (min shares needed to Shamir-reconstruct for `user-decrypt`) — which is exactly the `n − t = 3` number from a few questions back, not the bare `t+1 = 2` reconstruction minimum. 
> 	So: broadcast to all 4, but only `num_majority`/`num_reconstruct` of the replies actually have to come back correct.


### These are just local tests, moving to real machine for each node:
- DNS/addresses: swap abcd.dev-kms-core-N.com for real hostnames or IPs in each threshold.peers entry. No more /etc/hosts hack.
- Certs: --tls-subject matches the real hostname; ideally issued by a real/internal CA instead of the self-signed demo one.
- Network: the gRPC port and MPC port per node must be reachable between machines (firewall/security-group rules) — that's the main new operational concern, since on localhost everything's already reachable.
- Storage: you'd likely switch --private-storage/--public-storage from file to s3 (or similar shared storage), since each node's local disk isn't visible to the others anymore.

> [!note]
> Q: where are private keys and public keys stored? I know in local tests, they are just stored in local files, how about production case?
>
> A: Looked at `core/service/src/vault/` to check. Storage in this codebase is split into two independent layers:
> 1. **Where the bytes live** (`--private-storage`/`--public-storage`): `file` locally, or `s3` in production (`--private-s3-bucket`/`--public-s3-bucket`). Public keys always stay here as plain files/objects — no need to protect data that's meant to be public.
> 2. **Whether the bytes are protected before they're written** (a separate "keychain" concept, only applies to *private* data): locally, no keychain is set, so private keys are just written as plaintext files. In production, you'd attach an AWS KMS keychain (`--root-key-id`/`--root-key-spec`) — private key material gets envelope-encrypted before it's written to the private S3 bucket, so what's actually stored there is ciphertext, not the raw key. Unwrapping it requires calling AWS KMS at read time, which can itself be gated behind Nitro Enclave attestation (the `mock_enclave`/`security_module` flags I noticed earlier in `kms-gen-keys.rs` — proving "this request really comes from my approved enclave code" before AWS KMS will decrypt).
> 3. There's also a **custodian/secret-sharing keychain** option (`vault/keychain/secretsharing.rs`) — splits private data into shares held by separate custodians for disaster-recovery backup, so no single custodian can reconstruct it alone.
> So: local test = plaintext file, production = encrypted-at-rest (S3 + KMS envelope encryption, optionally enclave-gated), with an extra custodian-backup option layered on top.

**Check-in:** clear on why threshold nodes need mTLS certs (node-to-node trust) on top of the signing key each node already has (client-facing trust)?

---

## Chunk 6 — Running the client: KeyGen

With server(s) up, open a **new terminal** for `kms-core-client`. Everything below is the centralized-mode command; threshold mode uses the same commands, just pointed at a different config file (`client_local_threshold.toml` instead of `client_local_centralized.toml`).

KeyGen is a **two-step handshake**, not one command:

**Step 1 — Preprocessing:**
```bash
cargo run --bin kms-core-client -- \
  -f core-client/config/client_local_centralized.toml \
  insecure-preproc-key-gen
```
Output includes a `request_id` — call it `PREPROC_ID`, you'll need it in Step 2.

**Step 2 — KeyGen:**
```bash
cargo run --bin kms-core-client -- \
  -f core-client/config/client_local_centralized.toml \
  insecure-key-gen --preproc-id <PREPROC_ID>
```
Output includes another `request_id` — call it `KEY_ID`, you'll need it for both Encrypt and Decrypt later.

Why two steps instead of one? This maps directly to what we covered in Chunks 4-5: **Preprocessing** is where the heavy/collaborative setup happens 
(
* in threshold mode: the real MPC work — secret-key shares, noise, Beaver triples;
* in centralized mode: currently just a placeholder, no real work
)
**KeyGen** is the step that actually produces the usable public key + registers a private key(-share), consuming whatever Preprocessing set up. Splitting them lets the expensive Preprocessing work be done ahead of time, so the online KeyGen call is fast.

> [!note]
> Q: Yes or No: the KeyGen stage returns public key to client, and shares private key to nodes
>
> A: **Private key part: Yes.** It's written to each node's own private storage as a share — never sent to the client, never leaves the node.
> **Public key part: not quite** — KeyGen's gRPC response to the client is just the `KEY_ID` handle (that `request_id` you saw in the output), not the actual public key bytes. The public key material itself gets written to *public storage* by the node(s); the client fetches it separately, later, only when it actually needs to encrypt — that's the "reads directly from storage" behavior from Chunk 7's Q&A. So: KeyGen doesn't push the public key to the client at all; it just gives the client a name (`KEY_ID`) to go look it up by, whenever it's needed.

(The `insecure-` prefix is just this codebase's naming for a dev/test-mode command variant — not something to worry about yet.)

**Check-in:** does the Preprocessing/KeyGen split make sense as "prep the expensive stuff first, then do the fast final step," given what centralized vs. threshold mode each actually do during Preprocessing? Yes

---

## Chunk 7 — Running the client: Encrypt

```bash
cargo run --bin kms-core-client -- \
  -f core-client/config/client_local_centralized.toml \
  encrypt \
  --to-encrypt 0500000000000000 \
  --data-type euint64 \
  --key-id <KEY_ID> \
  --ciphertext-output-path ./ciphertext.bin
```
- `--to-encrypt 0500000000000000` — the number `5`, written as 8 bytes in little-endian hex.
- `--data-type euint64` — "encrypted unsigned 64-bit integer"; tells the client how to interpret/pad those bytes.
- `--key-id <KEY_ID>` — the id you got back from Chunk 6's KeyGen step.
- Output: a binary ciphertext blob written to `./ciphertext.bin`.

**The important part: this command never talks to `kms-server` at all.** The client just reads the *public* key (identified by `KEY_ID`) straight from storage and encrypts locally — no gRPC call, no server involved. That's possible because of plain public-key cryptography: anyone holding the public key can encrypt; only whoever holds the matching secret key (the server, or the threshold nodes' shares) can ever decrypt it. This is also why centralized and threshold clients run the *exact same* encrypt command — the public key is one combined artifact either way, so there's nothing mode-specific here.

> [!note]
> Q: The client just reads the *public* key (identified by `KEY_ID`) straight from storage and encrypts locally -> the client saves the public key in its local storage? where?
>
> A: Not "saves" — it *reads directly from whatever storage the server itself published to*, same as Chunk 5's private/public storage distinction. Traced it in `core/service/src/testing/utils.rs` (`compute_cipher_from_stored_key`, used by these `insecure-*` CLI commands): it opens a `FileStorage` pointed at the `./keys/PUB/` folder and loads the key file named by `KEY_ID`. In this local walkthrough that works simply because client, server, and `kms-gen-keys` are all the same machine sharing one filesystem — there's no copying/caching step, the client is reading the exact same folder the server wrote into during KeyGen.
>  * In a real deployment the client isn't on the same machine/disk as the server, so it can't do that — that's what the `s3_endpoint`/`object_folder` fields in `client_local_centralized.toml` are for (you may have noticed them): the client instead fetches the public key material over plain HTTP from the S3-compatible bucket the server publishes its public artifacts to. Same idea (read the server's public storage), different transport (local file vs. S3-over-HTTP) depending on deployment.

> [!note]
> This is for centralized encryption, is the threshold encryption the same?
>
> A: Yes, verified in `core-client/src/lib.rs`'s `CCCommand::Encrypt` handler — it calls `fetch_keys_auto_detect(...)` then `encrypt(...)`, with **no branch on `cc_conf.kms_type`** at all. Same code path runs whether the config says `centralized` or `threshold`.
> The only thing that differs is bookkeeping, not logic: `fetch_keys_auto_detect` has to figure out *which* configured core's storage actually holds this `KEY_ID`'s public key material — trivial in centralized (only one `[[cores]]` entry), a small search across the 4 party configs in threshold. And whichever core it resolves to, that core's storage backend is whatever was configured for it (plain file locally for centralized, S3-mock for threshold per the storage-location table above) — but that's a storage-backend detail the shared `encrypt()` call doesn't care about. So: same command, same function calls, same "never talks to `kms-server`" property either way — just a slightly bigger lookup step first in threshold mode.
>
> Follow-up Q: is the public key the same for threshold and centralized mode? And if so, why does the client need to *search* across 4 party configs — isn't it stored in one place?
>
> A: The public key is the same **content** in both modes (same `(A,b)` pair + BK + KSK — see the Chunk 11 correction above for how threshold nodes converge on that one identical value). But "same content" doesn't mean "stored in one place": after threshold KeyGen finishes, **each of the 4 nodes independently writes its own copy** of that identical public key to its own storage (`PUB-p1`, `PUB-p2`, `PUB-p3`, `PUB-p4` — same content, 4 separate locations, per the storage-location table from a few questions back). So there are 4 identical copies, not 1.
> Checked the actual search logic in `core-client/src/s3_operations.rs` (`fetch_public_elements`): it loops over `sim_conf.cores` **in order** and tries each one, but stops at the **first core that responds successfully** (`download_all = false` from `fetch_keys_auto_detect`, so the loop `break`s as soon as one core has everything). So the "search across 4 party configs" isn't hunting for *different* data — it's a **fallback for availability**: try node 1's storage; if that's unreachable, fall back to node 2's copy, then node 3's, then node 4's. Any one of the 4 would give the identical answer. In centralized mode there's only 1 core configured, so this fallback loop trivially has nothing to fall back to.


**Check-in:** does it make sense why Encrypt is the one step in this whole pipeline that needs zero interaction with `kms-server`? Yes

---

## Chunk 8 — Running the client: Decrypt (centralized mode)

```bash
cargo run --bin kms-core-client -- \
  -f core-client/config/client_local_centralized.toml \
  public-decrypt from-file \
  --input-path ./ciphertext.bin
```
- Reads the ciphertext blob from Chunk 7's `--ciphertext-output-path`.
- **This one *does* contact `kms-server`** — the opposite of Encrypt. The client sends the ciphertext over gRPC; the server uses the secret key to decrypt it and sends the plaintext back.

Expected output:
```
plaintexts: [TypedPlaintext { bytes: [5, 0, 0, 0, 0, 0, 0, 0], fhe_type: 5 }]
```
`bytes: [5, 0, 0, 0, 0, 0, 0, 0]` = `5` in little-endian — matching what you encrypted in Chunk 7. Round trip complete: Encrypt (client-only) → Decrypt (server-only) → same number back out.

This command is called `public-decrypt` specifically — there's a second variant, `user-decrypt`, that behaves the same for centralized mode, and differently in threshold mode (private results only the requesting client can see, vs. `public-decrypt` where every node learns the plaintext). That distinction only matters once multiple nodes are involved, so it's a topic for the threshold math chunks, not now.

> [!note]
> Q: so, this is centralized mode? if so, please update the chunk title, and generate note for the threshold case in next chunk
>
> A: Yes — that command (and Chunks 6-7's KeyGen/Encrypt) is centralized mode, using `client_local_centralized.toml`. Retitled this chunk to say so; threshold decrypt gets its own chunk right below, since it's the one command whose *behavior*, not just its config file, actually changes between modes.

**With this, you've run the centralized pipeline once end-to-end**: compile → run server → KeyGen → Encrypt → Decrypt. Threshold decrypt is next, then everything after that goes deeper into *why* it all works (the math), not new commands to run.

---

## Chunk 9 — Running the client: Decrypt (threshold mode)

Same CLI, pointed at the threshold config instead:
```bash
cargo run --bin kms-core-client -- \
  -f core-client/config/client_local_threshold.toml \
  public-decrypt from-file \
  --input-path ./ciphertext.bin
```
Same command shape as Chunk 8, same expected output — but what happens *behind* that one command is completely different, because no single node holds the whole secret key.

> [!note]
Q: The threshold case has `public-decrypt` and `user-decrypt`?
>
> A: Checked `core-client/src/lib.rs` — both `PublicDecrypt` and `UserDecrypt` are CLI commands available in **both** modes, not threshold-only. The difference is what they *mean*: in centralized mode (`client_local_centralized.toml` has `num_parties = 1`), `user-decrypt` still runs — one server signcrypts its full result under your ephemeral key — but there's no real secret-sharing or reconstruction happening, since there's only one party to begin with; it's the same command, just trivial. It only becomes the multi-party protocol described above (partial decryptions from several nodes, client reconstructing from `t+1` of them) once you're actually in threshold mode with multiple nodes holding separate shares.

**What actually happens across the 4 nodes:**
1. Every node computes a **partial decryption** using only its own key-share (not the full key — it doesn't have that).
2. Each node masks its partial result with a large pre-shared random value, so that one partial result alone reveals nothing useful on its own (this masking trick is called **noise-flooding** — its own topic for a later math chunk).
3. Then it branches into the two decrypt variants mentioned in Chunk 8:
   - **`public-decrypt`** (what's shown above): nodes run an extra "open" step so all of them converge on the same plaintext and each signs it. The client just takes one signed response — simplest case, but the result is visible to every node, so only use it when that's fine (e.g. a result meant to become public anyway).
   - **`user-decrypt`**: no open step. Each node instead encrypts *its own* masked partial result under the requesting client's one-time public key and sends that back. The client collects at least `t+1` of these, decrypts each with its own private key, and reconstructs the final plaintext **itself** — so no node (and nobody watching the network) ever sees the plaintext except the client. This is the mode you'd use for private results.

Both still end with you getting `5` back — the difference is entirely about *who else learns the plaintext along the way*, not the final answer.

**Check-in:** does the public-decrypt vs. user-decrypt distinction make sense as "does the client trust the network with the answer, or reconstruct it privately itself"? Ready to move into the math next (starting with `b = As + e`)? Yes.

---

## Chunk 10 — The math: LWE (Learning With Errors)

Everything you ran in Chunks 4-9 is built on one hardness assumption, **LWE**:

$$ \mathbf{b} = \mathbf{A}\mathbf{s} + \mathbf{e} \pmod q $$

| Symbol | What it is | Public or secret? |
|---|---|---|
| $\mathbf{s}$ | the secret key — a vector where every entry is **binary**, $s_i \in \{0,1\}$ | secret |
| $\mathbf{A}$ | a random matrix (the "mask") | public |
| $\mathbf{e}$ | small random noise | secret, but small enough that it doesn't change the answer once decrypted |
| $\mathbf{b}$ | $= \mathbf{A}\mathbf{s} + \mathbf{e}$ — looks like random noise to anyone who doesn't know $\mathbf{s}$ | public |

$(\mathbf{A}, \mathbf{b})$ together **are** the public key you used in Chunk 7 to encrypt. The **LWE problem** is: given only $(\mathbf{A}, \mathbf{b})$, recover $\mathbf{s}$. This is believed to be computationally hard — even for a quantum computer — which is exactly why it's safe to publish $(\mathbf{A}, \mathbf{b})$ but not $\mathbf{s}$.

**Why does $\mathbf{s}$ have to be binary** ($\{0,1\}$), not just "any small numbers"? Two payoffs used throughout this codebase:
1. **No multiplication needed** when the key gets used later (decryption, key-switching): if $s_i = 0$, skip; if $s_i = 1$, add. Much cheaper than a real polynomial multiply.
2. **Noise control**: small, sparse coefficients keep the noise growth predictable, so decryption doesn't fail from accumulated error.

This single equation is *the* thing Chunk 6's KeyGen actually produces (well — one of three artifacts; more on that next chunk). And Chunk 4's centralized vs. Chunk 5's threshold mode differ only in **who computes which of these four symbols, and when** — which is the natural next chunk.

> [!note]
> Q: Please tell me what quantity is produced at what stage. I remember that there's ECDSA signing keys, private keys, public keys ... etc. and there's KeyGen, Encrypt, Decrypt stage.
>
> A: Full inventory, in order of when each thing first gets produced:
>
> | Stage | Quantity | What it's for | Public/Secret | Lifetime |
> |---|---|---|---|---|
> | `kms-gen-keys` (Chunk 4/5, once, *before* any FHE key exists) | ECDSA signing key pair | proves "this response really came from this server/party" — signs everything KeyGen produces later | pub verify key / priv signing key | persists until rotated |
> | `kms-gen-keys threshold` only | mTLS certificate | node-to-node identity for the MPC network (Chunk 5) | cert is public, its key is private | persists |
> | KeyGen (Chunk 6) | $\mathbf{s}$ — secret key, binary | the actual FHE decryption key | **secret** | persists — needed for every future Decrypt |
> | KeyGen | $\mathbf{e}$ — noise | folded into $\mathbf{s}$'s protection, not used standalone | secret | persists (bundled with $\mathbf{s}$) |
> | KeyGen | $\mathbf{A}$ — random mask | half of the public key | public | persists |
> | KeyGen | $\mathbf{b} = \mathbf{A}\mathbf{s}+\mathbf{e}$ | the other half of the public key — what Chunk 7's Encrypt reads | public | persists |
> | KeyGen | Bootstrap Key + Key-Switching Key | two more artifacts alongside $(\mathbf{A},\mathbf{b})$, needed for the FHE machinery to keep working after ciphertext operations — **next chunk covers these properly** | public | persists |
> | Encrypt (Chunk 7) | the ciphertext | *uses* $(\mathbf{A},\mathbf{b})$, produces your `ciphertext.bin` — not new key material, just data | — | yours to keep/send |
> | Decrypt, `user-decrypt` only (Chunk 9) | client's ephemeral key pair | lets each node signcrypt its partial result just for you | both, but scoped to one request | generated fresh per request, discarded right after (this is the "ephemeral, not persisted" case from a few questions back) |
>
> So: signing keys are a one-time identity layer that has nothing to do with FHE math; $\mathbf{s}, \mathbf{e}, \mathbf{A}, \mathbf{b}$ + BK/KSK are all produced together at KeyGen and all persist long-term; Encrypt and Decrypt only *consume* that material (plus one throwaway ephemeral key pair in user-decrypt) — neither stage produces anything new that outlives the request.
>
> **Where is each of these actually stored?** Checked the real config files (`core/service/config/*.toml`) instead of assuming — one result was a genuine surprise (flagged below):
>
> | Quantity | Centralized, local test | Centralized, production | Threshold, local test | Threshold, production |
> |---|---|---|---|---|
> | ECDSA signing key + (threshold) mTLS cert | plain file, `./keys/PRIV` (Chunk 4's `kms-gen-keys --private-storage file`) | S3 bucket + AWS KMS envelope encryption (same `--root-key-id` flag from Chunk 5's Q&A, applied to `kms-gen-keys` here too) | plain file, `./keys/PRIV` per party (Chunk 5's `kms-gen-keys` command) | S3 bucket + AWS KMS envelope encryption, per party |
> | Public FHE material — $\mathbf{A}, \mathbf{b}$, BK, KSK | plain file, `./keys/PUB` (`default_centralized.toml`) | S3 bucket `zama_kms_public_keys`, plaintext — it's public, nothing to encrypt (`default_centralized_enclave.toml`) | **S3 (not file!)** — bucket `kms`, prefix `PUB-p1`/`PUB-p2`/... one prefix per node, plaintext (`compose_1.toml`, against a local S3-mock container) | S3 bucket, one prefix (or bucket) per node, plaintext |
> | Private FHE material — $\mathbf{s}, \mathbf{e}$ / each node's share | plain file, `./keys/PRIV` (`default_centralized.toml`) — literally unencrypted on disk | S3 bucket `zama_kms_private_keys` + AWS KMS envelope encryption (`[private_vault.keychain.aws_kms]`, `root_key_id = zama_kms_storage_root_key`), likely inside a Nitro Enclave (`default_centralized_enclave.toml`) | **S3 (not file!)**, prefix `PRIV-p1`/`PRIV-p2`/... — *no keychain configured*, so plaintext even though it's S3 (`compose_1.toml`; matches `mock_enclave = true` at the top — this is a dev/test config) | S3 bucket + AWS KMS envelope encryption per node — ideally each node's bucket/key in a *different* AWS account/operator, per the "isn't this dangerous" answer from earlier. (No ready-made example config for this exact case in the repo; inferred by analogy to the centralized-enclave file.) |
> | Ciphertext (`ciphertext.bin`) | wherever `--ciphertext-output-path` points, on whatever machine ran `encrypt` | same | same | same |
> | Ephemeral client key (user-decrypt) | in-memory only, in `kms-core-client`'s process, discarded after the request | same | same | same |
>
> The genuine surprise: **threshold's "local test" already uses S3 storage**, not plain files (`compose_1.toml` points at a local S3-mock container, `dev-s3-mock:9000`) — only centralized's local test (`default_centralized.toml`) uses plain files. So "local vs. production" for threshold is really "S3-mock, no encryption" vs. "real S3, AWS-KMS-encrypted" — the storage *backend* doesn't change between threshold's local and prod, only whether it's a mock and whether a keychain wraps the private data. Ciphertexts and the ephemeral user-decrypt key are the only two things that are always client-side, in all four combinations — never part of the KMS's own storage at all.
>
> **Correction — "one prefix per node" needs to be read carefully.** Checked `compose_2/3/4.toml` alongside `compose_1.toml`: all 4 nodes point at the **exact same** `s3_endpoint = "http://dev-s3-mock:9000"` and the **exact same** `bucket = "kms"` — the only thing that differs per node is the *prefix* (`PUB-p1`/`PRIV-p1`, `PUB-p2`/`PRIV-p2`, ...), basically a subfolder inside one bucket. So threshold's local test is **1 shared MinIO instance (`quay.io/minio/minio`, confirmed in `docker-compose-core-base.yml`), 1 bucket, 4 prefixes** — not 4 separate storage locations as "one prefix per node" could be misread to mean.
> This matters beyond pedantry: if a *production* threshold deployment did the same thing — all 4 nodes' private shares in one shared S3 bucket, just different folders — you'd be right back to the single-point-of-failure problem from the "isn't this dangerous" question earlier. Compromise that one shared bucket and you get all 4 shares at once, which defeats the entire reason threshold mode splits the key up. This docker-compose setup shares one MinIO purely for local-test convenience (one container instead of four). In real production, each node's private prefix should live in **physically/organizationally separate storage** — different buckets, ideally different AWS accounts or even different operators — precisely so no single storage compromise exposes more than one node's share. The "Threshold, production" cell above already said this (per-node bucket/key in a different account); this is the concrete reason why that separation actually matters, not just a nice-to-have.

**Check-in:** does it make sense why $(\mathbf{A}, \mathbf{b})$ can be public while $\mathbf{s}$ can't — i.e. why LWE being "hard to invert" is the whole security foundation here? Yes.

---

## Chunk 11 — Centralized vs. Threshold KeyGen: who computes what

Same equation, same target output $(\mathbf{s},\mathbf{e},\mathbf{A},\mathbf{b})$ — the only thing that changes between modes is **who computes each symbol, and in which of Chunk 6's two steps (Preprocessing vs. KeyGen)**:

| Step | Centralized | Threshold (DKG) |
|---|---|---|
| $\mathbf{s}$, $\mathbf{e}$ | sampled directly by the one server, during **KeyGen** | generated as **Shamir secret shares** across the 4 nodes, during **Preprocessing** |
| $\mathbf{A}$ | built on the fly, during KeyGen | built on the fly, during KeyGen (it's public — no need to share it) |
| $\mathbf{b} = \mathbf{A}\mathbf{s}+\mathbf{e}$ | computed directly | each node computes its own **local partial value** from its own share — no Beaver-triple-style MPC needed, since this formula is linear | 
| Bootstrap Key | computed directly | needs a **real multi-party computation step** — the one exception, covered next chunk |

So Chunk 4's centralized Preprocessing being "just a placeholder, no real work" (Chunk 6) now makes sense: there's nothing to precompute when one server is just going to sample $\mathbf{s}, \mathbf{e}$ itself in a single step anyway. Threshold's Preprocessing is doing real cryptographic work up front (generating the *shares* of $\mathbf{s}$ and $\mathbf{e}$ across the 4 nodes) precisely so that the online KeyGen call can be fast.

> [!note]
> **Correction**: the table's $\mathbf{b}$ row is easy to misread as "each node ends up with its own separate answer, no communication at all." That's not right. Since $\mathbf{s}$ and $\mathbf{e}$ are Shamir-*shared*, a single node only holds a share $s_i$ — computing $\mathbf{A}\cdot s_i$ alone gives a meaningless partial value, not the real, single, identical $\mathbf{b}$ that every node (and eventually the client) needs to use. So "no MPC needed" specifically means **no Beaver-triple-style secret multiplication protocol is needed** (that's the expensive, interactive kind of MPC) — not "zero communication." The nodes still run one lightweight round where they combine their local partial values (a linear/Lagrange combination, the same kind of math Shamir reconstruction always uses) to converge on the one shared, identical $\mathbf{b}$. Cheap and linear, but not silent.
> This also answers a natural follow-up: is the resulting public key actually the same object in both modes? **Yes** — same `(A,b)` pair + Bootstrap Key + Key-Switching Key, same `tfhe-rs` struct types, regardless of which mode produced it. That's exactly why Chunk 7's Encrypt command runs the identical code path either way — nothing about the public key's shape reveals which mode generated it. (What differs is that in threshold mode, each of the 4 nodes ends up separately persisting its own copy of that one identical public key — 4 copies of the same content, not 4 different keys — which is also why Encrypt's key-lookup step has more places to check, per the Chunk 7 Q&A above.)

The one row marked "real MPC step" — the Bootstrap Key — is the only place in this whole flow where nodes have to interact to compute something, rather than just working alone on their own share. That's genuinely the most interesting part, and it's next.

**Check-in:** does it make sense why $\mathbf{b}=\mathbf{A}\mathbf{s}+\mathbf{e}$ needs *no* coordination between threshold nodes (each just computes it locally from its own share), while the Bootstrap Key apparently does?

