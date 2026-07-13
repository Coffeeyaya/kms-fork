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

## Current Progress (as of 2026-07-13)

| Layer | Status | Summary |
|---|---|---|
| Layer 0 — Concept Map | ✅ Done | Centralized vs. Threshold, 5 actors |
| Layer 1 — Compile | ✅ Done | System deps, `cargo build`, what `target/` contains |
| Layer 2 — Run Server | ✅ Done | Centralized (1-server) + Threshold (4-node) setup |
| Layer 3 — Run Client | ✅ Done | KeyGen Phases 1-3: preproc → keygen → encrypt → decrypt |
| Layer 4 — Deep Dive | 🔍 In progress | KeyGen internals (centralized + threshold DKG) |

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
├── agent-instructions.md    ← THIS FILE
├── tutorial.md              ← Main learning journal (the user reads and writes here)
├── keygen_deep_dive.md      ← Layer 4: KeyGen math, code call traces, Mermaid diagrams
└── raw/                     ← Original reference notes (read-only)
    ├── kms-keygen-simple.md          Beginner story of centralized keygen
    ├── kms-keygen-deep-dive.md       Detailed function-by-function centralized trace
    ├── lwe-math-to-code-mapping.md   LWE equation → Rust struct mapping
    ├── 01_keygen_threshold.md        Threshold DKG flow overview
    └── kms-folder-and-run-overview.md  Repo layout + run overview
```

---

## Suggested Next Topics

Ask the user which they want to explore next:

- **A. Encryption deep dive** — how `b = As + e` enables client-side encryption (the math of `u = A^T r + e1`, `v = b^T r + e2 + Encode(m)`)
- **B. Decryption deep dive** — how the server uses `s` to recover plaintext (`v - s^T u ≈ Encode(m)`)
- **C. Threshold DKG math** — TFHE secret-key sharing, Beaver Triples for the Bootstrap Key's GGSW step, how `key-gen` works in MPC detail
- **D. Continue from any `Q:` the user writes in `tutorial.md`**

---

## How to Help

1. **Start by reading** `notes/tutorial.md` to see the full learning tree and any new `Q:` blocks the user wrote.
2. **Answer questions** inline in `tutorial.md` as `> Q: ... / > A: ...` blocks right below the relevant section.
3. **For deep dives**, create a new `notes/<topic>.md`, link it from `tutorial.md`, and keep it concise.
4. **Follow the confirmation rule**: after explaining a layer, present a simple checkpoint question and wait for the user to answer before going deeper.
5. **Mermaid diagrams**: use `graph TD` with `classDef` dark-mode CSS (see `notes/keygen_deep_dive.md` for the style reference).
