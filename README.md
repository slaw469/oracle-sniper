# Oracle Cloud Instance Sniper

Automated infrastructure provisioning system that monitors Oracle Cloud's free-tier ARM capacity across multiple availability domains and instantly provisions instances the moment resources become available — fully hands-off, running 24/7 via GitHub Actions.

## Problem

Oracle Cloud offers a generous Always Free tier (4 ARM CPUs, 24GB RAM), but demand far exceeds supply. Instances are nearly impossible to create manually — the console returns `Out of host capacity` errors indefinitely. Capacity opens in brief, unpredictable windows when other users release resources.

## Solution

A CI/CD-based sniper that runs every 5 minutes via GitHub Actions cron, cycling through all availability domains in a region. When capacity appears, it provisions the instance in seconds and triggers cloud-init for zero-touch server setup — no human intervention required at any point.

## Architecture

```
┌─────────────────────────────────────────────────┐
│                 GitHub Actions                   │
│              (cron: every 5 min)                 │
│                                                  │
│  ┌───────────┐  ┌───────────┐  ┌───────────┐   │
│  │   AD-1    │  │   AD-2    │  │   AD-3    │   │
│  │  Attempt  │─▶│  Attempt  │─▶│  Attempt  │   │
│  └───────────┘  └───────────┘  └───────────┘   │
│        │              │              │           │
│        ▼              ▼              ▼           │
│  ┌─────────────────────────────────────────┐    │
│  │         Capacity Available?             │    │
│  └──────────────┬──────────────────────────┘    │
│            yes  │  no                            │
│                 ▼                                 │
│  ┌──────────────────────┐    ┌──────────────┐   │
│  │  Launch Instance     │    │  Exit, retry  │   │
│  │  + Cloud-Init Setup  │    │  in 5 min     │   │
│  └──────────┬───────────┘    └──────────────┘   │
│             ▼                                    │
│  ┌──────────────────────┐                       │
│  │  Create GitHub Issue  │                       │
│  │  with IP + SSH info   │                       │
│  └──────────────────────┘                       │
└─────────────────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────┐
│              Oracle Cloud Instance               │
│                                                  │
│  Cloud-Init (runs on first boot):                │
│  ├── System update                               │
│  ├── Node.js 20                                  │
│  ├── Claude Code CLI                             │
│  ├── Dev tools (git, tmux, htop, jq)             │
│  └── UFW firewall (SSH only)                     │
└─────────────────────────────────────────────────┘
```

## Features

- **Automated capacity monitoring** — polls Oracle Cloud API every 5 minutes across all availability domains in the region
- **Zero-touch provisioning** — cloud-init script handles full server setup on first boot (Node.js, Claude Code, dev tools, firewall)
- **Duplicate prevention** — checks for existing running instances before each attempt to avoid accidental resource duplication
- **Rate limit handling** — detects API throttling and backs off gracefully
- **GitHub Issue notification** — creates a detailed issue with the public IP, SSH command, and setup status when an instance is successfully provisioned
- **Fully serverless** — runs entirely on GitHub Actions, no local machine required

## How It Works

1. **Cron trigger** — GitHub Actions fires the workflow every 5 minutes
2. **Existence check** — queries OCI API for any running instance named `claude-server` to prevent duplicates
3. **Multi-AD sweep** — attempts to launch a `VM.Standard.A1.Flex` (4 OCPU, 24GB RAM) instance in each availability domain sequentially
4. **Capacity detection** — parses API responses to distinguish between capacity errors (expected, retry later) and real errors
5. **IP retrieval** — on successful launch, polls for the public IP assignment (up to 10 attempts)
6. **Cloud-init bootstrap** — the instance self-configures on first boot using an embedded cloud-init script
7. **Notification** — creates a GitHub Issue with connection details

## Setup

### Prerequisites

- Oracle Cloud account (free tier)
- GitHub account
- OCI CLI API key configured

### GitHub Secrets Required

| Secret | Description |
|--------|-------------|
| `OCI_USER_OCID` | Oracle Cloud user OCID |
| `OCI_FINGERPRINT` | API key fingerprint |
| `OCI_TENANCY_OCID` | Tenancy OCID |
| `OCI_REGION` | Home region (e.g., `us-chicago-1`) |
| `OCI_PRIVATE_KEY` | OCI API private key (PEM format) |
| `SSH_PUBLIC_KEY` | SSH public key for instance access |

### Deploy

```bash
# Clone the repo
git clone https://github.com/slaw469/oracle-sniper.git
cd oracle-sniper

# Set your secrets via GitHub CLI
gh secret set OCI_USER_OCID --body "ocid1.user.oc1..xxx"
gh secret set OCI_FINGERPRINT --body "aa:bb:cc:..."
gh secret set OCI_TENANCY_OCID --body "ocid1.tenancy.oc1..xxx"
gh secret set OCI_REGION --body "us-chicago-1"
gh secret set OCI_PRIVATE_KEY < ~/.oci/your_api_key.pem
gh secret set SSH_PUBLIC_KEY < ~/.ssh/your_key.pub

# Trigger manually to test
gh workflow run "Oracle Instance Sniper"

# Watch the run
gh run watch
```

### Configuration

Edit the environment variables in `.github/workflows/sniper.yml` to match your OCI setup:

```yaml
env:
  COMPARTMENT_ID: "your-tenancy-ocid"
  SUBNET_ID: "your-subnet-ocid"
  IMAGE_ID: "your-image-ocid"
  SHAPE: "VM.Standard.A1.Flex"
```

Update the availability domain list in the snipe step to match your region.

## Cloud-Init Bootstrap

The provisioned instance is production-ready on first boot:

| Component | Details |
|-----------|---------|
| **OS** | Ubuntu 24.04 LTS (ARM64) |
| **Runtime** | Node.js 20 LTS |
| **AI Tooling** | Claude Code CLI |
| **Dev Tools** | git, tmux, htop, jq, build-essential |
| **Security** | UFW firewall (SSH-only ingress) |

Setup progress can be monitored via `cat /var/log/server-setup.log` after SSH.

## Cost

$0. The entire pipeline runs on free infrastructure:

- **GitHub Actions** — free for public repositories
- **Oracle Cloud** — Always Free tier (4 ARM CPUs, 24GB RAM, 200GB storage, 10TB egress)

## Tech Stack

- GitHub Actions (CI/CD orchestration)
- OCI CLI (Oracle Cloud Infrastructure API)
- Cloud-Init (instance bootstrapping)
- Bash (automation scripting)

## License

MIT
