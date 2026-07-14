# Agent Instructions — KMS Learning Session

> Read this file first when starting a new chat to continue helping the user learn the Zama KMS codebase.

---

## Context

The user (**tsaiyunchen**) is learning the **Zama KMS (Key Management System)** codebase from scratch. They are new to FHE cryptography and MPC. The goal is a structured, tree-layered learning experience: confirm understanding at each layer before going one level deeper.

---

## Learning Approach

- **Tree-structured layers** — each layer is confirmed by the user before diving deeper.
- **Inline Q&A** — when the user asks a question, answer it directly in `tutorial.md` as a `> Q: ... / > A: ...` block, right below the relevant section.
- **Separate deep-dive files** — when a topic needs extensive exploration, create a new `notes/<topic>.md` and link it from `tutorial.md`.
- **All original reference notes** are archived in `notes/raw/` — do not edit those files.
- **Style**: concise, beginner-friendly, link to code files when relevant, dark-mode Mermaid diagrams with `classDef` CSS.

---

## Current Progress (as of 2026-07-14)

| Layer | Status | Summary |
|---|---|---|
| Layer 0 — Concept Map | ✅ Done | Centralized vs. Threshold, 5 actors |
| Layer 1 — Compile | ✅ Done | System deps, `cargo build`, what `target/` contains |
| Layer 2 — Run Server | ✅ Done | Centralized (1-server) + Threshold (4-node) setup |
| Layer 3 — Run Client | ✅ Done | KeyGen Phases 1-3: preproc → keygen → encrypt → decrypt |
| Layer 4 — Deep Dive | ✅ Done | KeyGen (centralized + threshold DKG) and Encrypt/Decrypt, each with their own file |

### Notes de-duplication pass (2026-07-14)
`tutorial.md` used to re-explain the LWE variable table, the Beaver-triple Q&A, and the binary-vs-ternary coefficient correction inline in Phase 1 — the same content that `keygen_deep_dive.md` / `threshold_dkg_deep_dive.md` cover in more depth. Trimmed those sections down to a short summary + links, so each fact now lives in exactly one file. Also wrote the previously-missing `notes/encrypt_decrypt_deep_dive.md` (existing docs linked to it, but it was never created) by merging the encrypt/decrypt content that was scattered across `tutorial.md` Phase 2/3 and `kms_crypto_notes.md` §4-5. `tutorial.md`'s own checkpoint `Ans:` lines (the user's own written answers) were left untouched — only the agent-authored explanatory `Q:/A:` blocks were trimmed.

### What was covered in Layer 3 (KeyGen):
- **LWE math**: `b = As + e`, and how `s`, `e`, `A`, `b` map to Rust structs (`ClientKey`, `FhePublicKey`, `ServerKey`)
- **Centralized mode**: all keys (s, e, A, b) generated in the KeyGen step; preprocessing is a dummy placeholder
- **Threshold mode**: `s` generated as MPC shares during preprocessing (real DKG); LWE public key `b = As + e` computed locally per party in KeyGen; Bootstrap Key and Key-Switching Key are separate artifacts (see correction below)
- **Key coefficients**: **binary**, `s_i ∈ {0, 1}` — eliminates multiplication, controls noise growth
- **Diagrams**: dark-mode flowcharts with `classDef` CSS in `notes/keygen_deep_dive.md`

### Correction made (2026-07-13)
`threshold_dkg_deep_dive.md` (and parts of `keygen_deep_dive.md` / `tutorial.md`) originally documented `core/threshold-bgv` — an **unused experimental BGV crate** — instead of the real threshold DKG path, `core/threshold-execution/src/endpoints/keygen.rs`. This produced two wrong claims that have since been fixed: (1) secret key coefficients are **binary** `{0,1}`, not ternary `{-1,0,1}`; (2) Beaver triples are needed only for the **Bootstrap Key's GGSW step** (multiplying a GLWE-key bit by an LWE-key bit), not for a BGV-style `s²` relinearization term, which doesn't exist in this codebase's real path. **If you're a future agent continuing this session: trust the corrected versions of these files, not `notes/raw/01_keygen_threshold.md` or `notes/raw/lwe-math-to-code-mapping.md` (the latter also has a minor, separate citation error — see the note in `keygen_deep_dive.md`'s intro). Both `raw/` files are read-only archive and were not edited.**

---

## Notes File Map

```
notes/
├── agent-instructions.md          ← THIS FILE (read first, session continuity)
├── tutorial.md                    ← Main learning journal: Layers 0-3 (concept map,
│                                     compile, run server, run client). The user's own
│                                     checkpoint Ans: are theirs — don't rewrite those.
├── kms_crypto_notes.md            ← Condensed one-page version of every deep dive below.
│                                     Read this for a fast, no-click-through pass.
├── keygen_deep_dive.md            ← Layer 4: KeyGen math, code call traces, Mermaid diagrams
│                                     (centralized + threshold, both modes in one file)
├── threshold_dkg_deep_dive.md     ← Layer 4: DKG preprocessing + KeyGen MPC, line-cited
├── encrypt_decrypt_deep_dive.md   ← Layer 4: encryption code path + threshold decrypt
│                                     (public-decrypt vs. user-decrypt), code-cited
└── raw/                           ← Original reference notes (read-only, do not edit)
    ├── kms-keygen-simple.md          Beginner story of centralized keygen
    ├── kms-keygen-deep-dive.md       Detailed function-by-function centralized trace
    ├── lwe-math-to-code-mapping.md   LWE equation → Rust struct mapping (minor errata,
    │                                 see keygen_deep_dive.md's Erratum note)
    ├── 01_keygen_threshold.md        Superseded/inaccurate threshold flow — kept for archive
    ├── 01_keygen_central.md          Early centralized keygen notes
    ├── 00_run.md                     Early run notes
    ├── kms-folder-and-run-overview.md  Repo layout + run overview
    ├── kms-run-centralized-on-codespace.md  Codespace-specific run notes
    ├── kms-docker-blocked-private-images.md Docker/private-image troubleshooting
    └── prism-learning-map.md         Original learning-plan brainstorm

Every top-level file links back to tutorial.md and links out to the next-deeper file —
follow the "Back to:" / "🔍 Deep Dive" lines to navigate the tree instead of guessing paths.
```

---

## Suggested Next Topics

Layers 0-4 (KeyGen, DKG, Encrypt/Decrypt) are now covered. Ask the user which they want next:

- **A. Go deeper on tfhe-rs internals** — the actual LWE encrypt/decrypt arithmetic (`u = A^T r + e1`, `v = b^T r + e2 + Encode(m)`) lives inside the `tfhe-rs` dependency, not this repo — could trace into that crate if the user wants it
- **B. Noise-flooding math** — why the mask needs to be "big enough" to hide a partial-decryption share, and how that bound is chosen
- **C. Continue from any `Q:` the user writes in `tutorial.md`**
- **D. Something outside KeyGen/Encrypt/Decrypt** — e.g. key rotation, custodian recovery, gRPC/mTLS session setup

---

## How to Help

1. **Start by reading** `notes/tutorial.md` to see the full learning tree and any new `Q:` blocks the user wrote.
2. **Answer questions** inline in `tutorial.md` as `> Q: ... / > A: ...` blocks right below the relevant section.
3. **For deep dives**, create a new `notes/<topic>.md`, link it from `tutorial.md`, and keep it concise.
4. **Follow the confirmation rule**: after explaining a layer, present a simple checkpoint question and wait for the user to answer before going deeper.
5. **Mermaid diagrams**: use `graph TD` with `classDef` dark-mode CSS (see `notes/keygen_deep_dive.md` for the style reference).
