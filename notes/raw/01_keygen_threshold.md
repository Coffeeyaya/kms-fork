### Focus Areas for Threshold Key Generation

To follow the journey of this multi-party data structure and witness how threshold keys are collaboratively generated, you should focus on these files:

1. **📂 Client Handler:** [core-client/src/keygen.rs](file:///Users/tsaiyunchen/Desktop/CSLAB/04_Work/kms/core-client/src/keygen.rs)
    - **What it does:** Packages and signs the request. Just like in centralized mode, the client must trigger a preprocessing step (`insecure-preproc-key-gen` / `preproc-key-gen`) to generate a preprocessing ID first. It then broadcasts the key generation command (`insecure-key-gen` / `key-gen`) along with the preprocessing ID to all the KMS cores, validating that enough valid matching responses are received to satisfy quorums.

2. **📂 Server Orchestrator:** [key_generator.rs](file:///Users/tsaiyunchen/Desktop/CSLAB/04_Work/kms/core/service/src/engine/threshold/service/key_generator.rs)
    - **What it does:** This is the server-side coordinator. It receives key-gen requests, verifies signature quorums, transitions the servers together through the Multi-Party Computation Distributed Key Generation (DKG) steps, and registers the session.

3. **📂 Cryptographic Protocols (MPC Math):** [dkg.rs](file:///Users/tsaiyunchen/Desktop/CSLAB/04_Work/kms/core/threshold-bgv/src/bgv/dkg.rs) & [dkg_orchestrator.rs](file:///Users/tsaiyunchen/Desktop/CSLAB/04_Work/kms/core/threshold-bgv/src/bgv/dkg_orchestrator.rs)
    - **What it does:** Implements the Multi-Party Computation math. It manages generating polynomials, exchanging encrypted evaluation shares between nodes, and checking commitments to ensure no party is cheating during setup.

4. **📂 Network Messenger (Inter-Node Communication):** [sending_service.rs](file:///Users/tsaiyunchen/Desktop/CSLAB/04_Work/kms/core/threshold-networking/src/sending_service.rs) & [grpc.rs](file:///Users/tsaiyunchen/Desktop/CSLAB/04_Work/kms/core/threshold-networking/src/grpc.rs)
    - **What it does:** Coordinates low-level mTLS protected inter-core communication channels over gRPC to swap protocol shares.

5. **📂 Storage Layer:** [threshold.rs](file:///Users/tsaiyunchen/Desktop/CSLAB/04_Work/kms/core/service/src/vault/storage/crypto_material/threshold.rs)
    - **What it does:** Saves the outputs. The public evaluation keys are unified and written to public storage, while each node saves its unique private secret key share to its own vault.

---

### How the TOML Parameters Change

To understand how the configuration parameters reflect this threshold workflow, compare the TOMLs:

- **On the Server Side ([compose_4.toml](file:///Users/tsaiyunchen/Desktop/CSLAB/04_Work/kms/core/service/config/compose_4.toml)):** Defines the node cluster topology (`[[threshold.peers]]`) where nodes list each other's addresses, ports, and certificates for authentication.
- **On the Client Side ([client_local_threshold.toml](file:///Users/tsaiyunchen/Desktop/CSLAB/04_Work/kms/core-client/config/client_local_threshold.toml)):** Dictates cluster consensus thresholds:
    - `kms_type = "threshold"`
    - `num_parties = 4` (Total nodes in the network cluster)
    - `num_majority = 2` (Minimum number of nodes required to guarantee an honest consensus quorum)
    - `num_reconstruct = 3` (Minimum number of shares required to decrypt or interact with the key)

---

### Summary of the Phase

When you invoke the threshold client `key-gen` command:
1. The client connects to all 4 cores.
2. The client triggers preprocessing via `insecure-preproc-key-gen` / `preproc-key-gen` to obtain a preprocessing ID.
3. The client triggers key generation via `insecure-key-gen` / `key-gen --preproc-id <ID>`.
4. Cores verify EIP-712 auth, orchestrate via `key_generator.rs`, perform MPC math in `dkg.rs`, communicate over `threshold-networking`, and write output shares using `threshold.rs`.