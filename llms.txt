# Deploy to EigenCompute

Two deployment methods: **Deploy from registry** (pre-built Docker image) or **Build from verifiable source** (EigenCompute builds from your GitHub repo inside the TEE for reproducible builds).

## Prerequisites

- Node.js 20+
- `ecloud` CLI: `curl -fsSL https://raw.githubusercontent.com/Layr-Labs/eigencloud-tools/master/install-all.sh | bash`
- Docker installed with buildx (registry method only)

---

## Step 1 — Authenticate

```bash
ecloud auth login
```

Stores your private key in OS keyring. Verify with:

```bash
ecloud auth whoami
```

---

## Step 2 — Set Up Billing

```bash
ecloud billing subscribe
```

Required before your first deploy.

---

## Step 3 — Create `.env`

Your `.env` file gets injected into the TEE at deploy time. Example:

```env
PORT=3000
OPENROUTER_API_KEY=sk-or-v1-...
SUPABASE_URL=https://xxx.supabase.co
SUPABASE_SERVICE_KEY=eyJ...
```

---

# Method A — Deploy from Registry

Use this when you want to build the Docker image yourself and push it to Docker Hub.

### A1 — Build Your App

```bash
npm run build
```

### A2 — Build & Push Docker Image

**Must be `linux/amd64`** — EigenCompute runs Intel TDX enclaves.

```bash
docker buildx build --platform linux/amd64 --no-cache \
  -t yourdockerhub/yourapp:v1.0.0 --push .
```

### A3 — Deploy

```bash
ecloud compute app deploy \
  --image-ref yourdockerhub/yourapp:v1.0.0 \
  --env-file .env \
  --instance-type g1-standard-4t \
  --log-visibility public \
  --resource-usage-monitoring enable
```

The CLI will interactively ask **7 prompts** in this exact order:

| # | Prompt | Answer |
|---|--------|--------|
| 1 | `Build from verifiable source? (y/N)` | **N** |
| 2 | `Enter Docker image reference:` | `yourdockerhub/yourapp:v1.0.0` |
| 3 | `Enter app name:` | Your app name (default: from image) |
| 4 | `Choose an option:` (env file) | `Enter path to existing env file` → `.env` **or** `Continue without env file` |
| 5 | `Choose instance:` | Pick from 6 options (see Instance Types) |
| 6 | `Do you want to view your app's logs?` | `Yes, publicly viewable by anyone` |
| 7 | `Show resource usage (CPU/memory) for your app?` | `Yes, enable resource usage monitoring` |

No port prompt, no confirmation prompt — deployment starts immediately after prompt 7.

### A4 — Upgrading

Build & push a new image tag, then:

```bash
ecloud compute app upgrade <APP-ID> \
  --image-ref yourdockerhub/yourapp:v1.0.1 \
  --env-file .env
```

---

# Method B — Build from Verifiable Source

Use this when you want EigenCompute to build your app directly from a GitHub repo. This gives you **reproducible, verifiable builds** — anyone can confirm the running code matches the source.

### Requirements

- Public GitHub repo
- Repo must contain a `Dockerfile`

### B1 — Deploy

```bash
ecloud compute app deploy --verifiable \
  --repo https://github.com/yourusername/yourapp \
  --commit <40-char-sha> \
  --env-file .env \
  --instance-type g1-standard-4t \
  --log-visibility public \
  --resource-usage-monitoring enable
```

The CLI will interactively ask **9 prompts** in this exact order:

| # | Prompt | Answer |
|---|--------|--------|
| 1 | `Build from verifiable source? (y/N)` | **y** |
| 2 | `Choose verifiable source type:` | `Build from git source (public repo required)` |
| 3 | `Enter public git repository URL:` | `https://github.com/yourusername/yourapp` |
| 4 | `Enter git commit SHA (40 hex chars):` | Full 40-char commit hash (get yours: `git rev-parse HEAD`) |
| 5 | `Enter build context path (relative to repo):` | `.` (default) |
| 6 | `Enter Dockerfile path (relative to build context):` | `Dockerfile` (default) |
| 7 | `Enter Caddyfile path (optional):` | Press Enter to skip |
| 8 | `Enter dependency digests:` | Press Enter to skip |
| 9 | `Choose an option:` (env file) | `Enter path to existing env file` → `.env` **or** `Continue without env file` |

No instance type, log, or monitoring prompts — verifiable builds use defaults. Use flags to customize.

Only **one verifiable build** can run at a time per account.

### B2 — Upgrading

Push changes to your repo, then redeploy with the new commit SHA:

```bash
ecloud compute app upgrade <APP-ID> --verifiable \
  --repo https://github.com/yourusername/yourapp \
  --commit <new-40-char-sha> \
  --env-file .env
```

---

## Verify

```bash
# Check status
ecloud compute app info --watch

# Stream logs
ecloud compute app logs --watch

# Health check
curl http://<your-ip>:3000/health
```

On success you get:
- **App ID** (e.g. `0x8F38007B...82A7`)
- **Public IP** (e.g. `35.222.154.28`)

---

## Useful Commands

| Command | What it does |
|---------|-------------|
| `ecloud auth whoami` | Show authenticated address |
| `ecloud compute app list` | List all your apps |
| `ecloud compute app info <APP-ID>` | App status & IP |
| `ecloud compute app logs <APP-ID> -w` | Stream logs |
| `ecloud compute app stop <APP-ID>` | Stop app |
| `ecloud compute app start <APP-ID>` | Start app |
| `ecloud compute app terminate <APP-ID>` | Delete app permanently |
| `ecloud billing status` | Check billing |

---

## Instance Types

| Type | Specs | TEE Technology |
|------|-------|---------------|
| `g1-micro-1v` | Shared 2 vCPUs, 1 GB memory | Shielded VM (default) |
| `g1-medium-1v` | Shared 2 vCPUs, 4 GB memory | Shielded VM |
| `g1-custom-2-4096s` | 2 vCPUs, 4 GB memory | AMD SEV-SNP |
| `g1-standard-2s` | 2 vCPUs, 8 GB memory | AMD SEV-SNP |
| `g1-standard-4t` | 4 vCPUs, 16 GB memory | Intel TDX |
| `g1-standard-8t` | 8 vCPUs, 32 GB memory | Intel TDX |

---

## Important Notes

- Docker images **must** be `linux/amd64` — ARM won't work
- Your app **must** listen on `0.0.0.0` (not `127.0.0.1`) — containers need external access
- Your app runs as `root` inside the TEE (required by EigenCompute)
- `.env` vars are encrypted and injected at runtime
- TEE provides hardware-level isolation via Intel TDX
- `g1-micro-1v` instances do **not** support TDX attestation — use `g1-standard-4t` or higher for production
- Match results/outputs can be cryptographically attested using the KMS key at `/usr/local/bin/kms-signing-public-key.pem`
- **Verifiable builds** (Method B) record the build hash on-chain, enabling third-party verification that the deployed code matches the source repo
