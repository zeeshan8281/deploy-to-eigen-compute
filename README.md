# Deploy to EigenCompute

Step-by-step guide to deploy any Dockerized app to [EigenCompute](https://docs.eigencloud.xyz/eigencloud/eigencloud-overview) — EigenLayer's verifiable compute platform running inside Intel TDX Trusted Execution Environments.

## Quick Start

```bash
# 1. Install CLI
curl -fsSL https://raw.githubusercontent.com/Layr-Labs/eigencloud-tools/master/install-all.sh | bash

# 2. Authenticate
ecloud auth login

# 3. Subscribe to billing
ecloud billing subscribe

# 4. Build & push Docker image (must be linux/amd64)
docker buildx build --platform linux/amd64 --no-cache -t yourdockerhub/yourapp:v1.0.0 --push .

# 5. Deploy
ecloud compute app deploy

# 6. Verify
ecloud compute app info --watch
```

## Full Guide

Open `index.html` in your browser for the complete interactive walkthrough, or view the [deployed version](https://zeeshan8281.github.io/deploy-to-eigen-compute/).

## What's Covered

- Installing the `ecloud` CLI
- Authentication with Ethereum private keys
- Billing setup
- Building your app & Dockerfile
- Building `linux/amd64` Docker images (required for Intel TDX)
- Interactive deployment flow with all CLI prompts explained
- Health checks & log streaming
- Upgrading existing deployments
- Full command reference

## Resources

- [EigenCloud Docs](https://docs.eigencloud.xyz/)
- [ecloud CLI — GitHub](https://github.com/Layr-Labs/ecloud)
- [eigencloud-tools Installer](https://github.com/Layr-Labs/eigencloud-tools)
- [EigenCloud Developer Console](https://console.eigencloud.xyz/)

## Verified Against

`ecloud-cli v0.3.4` — every command, flag, and option in this guide has been verified against the actual CLI `--help` output.
