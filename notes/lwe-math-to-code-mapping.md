# LWE Cryptography: Mathematical Derivation & Code Realization

This note maps the mathematical foundation of Learning With Errors (LWE) encryption and decryption directly to their concrete realizations in the Zama KMS Rust codebase.

**Related notes:**
- [KMS Folder & Run Overview](./kms-folder-and-run-overview.md)
- [Key Generation Deep Dive](./kms-keygen-deep-dive.md)

---

## 1. Key Generation
* **Mathematical Formula**:
  $$\mathbf{b} = \mathbf{A}\mathbf{s} + \mathbf{e} \pmod q$$
  Where $\mathbf{s}$ is the secret vector, $\mathbf{A}$ is a public matrix, and $\mathbf{e}$ is a small error/noise vector.
* **Code Realization**:
  * Generated in `kms-gen-keys.rs` and the `key_gen_impl` function.
  * In the codebase, the secret key $\mathbf{s}$ is stored as a `ClientKey` (contained inside `KmsFheKeyHandles` in [central_kms.rs](../kms/core/service/src/engine/centralized/central_kms.rs#L674)).
  * The public key $(\mathbf{A}, \mathbf{b})$ is stored as a `FhePublicKey` or compressed as a `CompressedXofKeySet` (see [test_tools.rs](../kms/core/service/src/util/key_setup/test_tools.rs#L349)).
  * See [Key Generation Deep Dive](./kms-keygen-deep-dive.md) for the full call stack.

---

## 2. Encryption (Offline)
* **Mathematical Formula**:
  The client encrypts message $m$ using a random vector $\mathbf{r}$ and small noise terms $\mathbf{e_1}, e_2$:
  $$\mathbf{u} = \mathbf{A}^T\mathbf{r} + \mathbf{e_1} \pmod q$$
  $$v = \mathbf{b}^T\mathbf{r} + e_2 + \text{Encode}(m) \pmod q$$
  Where:
  $$\text{Encode}(m) = m \times \frac{q}{p}$$
* **Code Realization**:
  * Done offline by the client in [lib.rs](../kms/core-client/src/lib.rs#L1536) (under the `encrypt` and `compute_cipher` functions).
  * The encoding and encryption is executed by `expanded_encrypt(pk, msg, num_bits)` in [test_tools.rs](../kms/core/service/src/util/key_setup/test_tools.rs#L44).
  * The client only takes `FhePublicKey` as an argument. The secret key is not accessed.

---

## 3. Decryption (Online)
* **Mathematical Formula**:
  The server removes the public key mask by evaluating the ciphertext against the secret key $\mathbf{s}$:
  $$\text{Result} = v - \mathbf{s}^T\mathbf{u} \pmod q$$
  By expanding $\mathbf{b}^T = \mathbf{s}^T\mathbf{A}^T + \mathbf{e}^T$:
  $$v - \mathbf{s}^T\mathbf{u} = \big(\mathbf{s}^T\mathbf{A}^T\mathbf{r} + \mathbf{e}^T\mathbf{r} + e_2 + \text{Encode}(m)\big) - \mathbf{s}^T\big(\mathbf{A}^T\mathbf{r} + \mathbf{e_1}\big)$$
  $$v - \mathbf{s}^T\mathbf{u} = \text{Encode}(m) + \underbrace{\mathbf{e}^T\mathbf{r} + e_2 - \mathbf{s}^T\mathbf{e_1}}_{\text{noise}}$$
  Since the noise is small, rounding the result to the nearest multiple of $q/p$ yields $m$.
* **Code Realization**:
  * Implemented in the centralized server under `unsafe_decrypt` in [central_kms.rs](../kms/core/service/src/engine/centralized/central_kms.rs#L659).
  * Evaluated as:
    ```rust
    let raw_res = hl_ct.decrypt(&keys.client_key);
    ```
    Where `keys.client_key` is the secret key $\mathbf{s}$, and `decrypt` performs the dot product subtraction and rounding.
