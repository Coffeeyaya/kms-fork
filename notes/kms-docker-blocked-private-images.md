# KMS: Docker Centralized Mode on GitHub Codespace

> Attempt to run KMS server + client on Codespace via prebuilt Docker images.  
> RAM required: ~5–6 GB → works on 8 GB+ Codespace.

---

## What Would Run

```
Codespace Terminal 1 (server):
  docker compose up
    ├── dev-s3-mock        (MinIO S3 mock — ports 9000, 9001)
    ├── dev-s3-mock-setup  (one-shot setup, then exits)
    └── dev-kms-core       (KMS server — port 50051)

Codespace Terminal 2 (client):
  docker run kms-core-client → connects to localhost:50051
```

---

## Step 1: Open Codespace

Go to `https://github.com/Coffeeyaya/kms`, click:  
**Code → Codespaces → Create / Resume codespace**

> Use machine type: **4-core / 8 GB RAM** or larger.

Once inside, open the terminal. All commands below run there.

---

## Step 2: Navigate to Repo Root

```bash
cd /workspaces/kms
```

---

## Step 3: Authenticate to `ghcr.io`

Generate a GitHub PAT:
- Go to: https://github.com/settings/tokens
- Click **Generate new token (classic)**
- Scope: ✅ `read:packages` only
- Copy the token

Login in Codespace terminal:
```bash
docker login ghcr.io -u <your-github-username> -p <your-PAT>
```

Expected: `Login Succeeded`

---

## Step 4: Pull the Images

```bash
docker pull ghcr.io/zama-ai/kms/core-service:latest-dev
docker pull ghcr.io/zama-ai/kms/core-client:latest
docker pull quay.io/minio/minio
docker pull quay.io/minio/mc
```

---

## ❌ Blocked Here — Private Images

Even after a successful `docker login`, pulling `ghcr.io/zama-ai/kms/core-service:latest-dev` returns:

```
403 Forbidden
```

![403 Forbidden error screenshot](./Pasted%20image%2020260711151142.png)

**Root cause**: The `latest-dev` image is a **private package restricted to Zama org members**.  
A GitHub PAT with `read:packages` scope only grants access to packages your account has permission to read.  
External contributors / forks cannot access these images.

Attempting to build locally also fails:

```
failed to solve: failed to fetch anonymous token: unexpected status from GET request to
https://cgr.dev/token?scope=repository%3Azama.ai%2Fglibc-dynamic%3Apull&service=cgr.dev: 401 Unauthorized
```

The Dockerfile uses `cgr.dev/zama.ai/glibc-dynamic` (Chainguard) as a base image — also a private registry requiring Zama org credentials.

---

## Conclusion

| Approach | Status | Reason |
|---|---|---|
| Docker prebuilt images (`ghcr.io`) | ❌ | Private — Zama org members only |
| Docker local build | ❌ | Requires Chainguard (`cgr.dev`) access |
| Native Cargo build on Codespace | ✅ | Works — see `workspace_analysis.md` |

**The only working path for external contributors is building from source natively.**  
See [kms-run-centralized-on-codespace.md](./kms-run-centralized-on-codespace.md) for the verified native Cargo workflow.
