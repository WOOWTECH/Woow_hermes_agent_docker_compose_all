# Woow Hermes Agent — K3s Kubernetes Deployment

**Hermes AI Agent on K3s — Production Deployment Guide**
**Hermes AI 智慧助理 K3s — 生產環境部署指南**

This branch contains the complete K3s Kubernetes deployment for Hermes AI smart home assistant, including custom Docker image with 26 CLI tools, RBAC, Cloudflare Tunnel, and enterprise-grade test suite (80+ tests).

此分支包含 Hermes AI 智慧助理的完整 K3s 部署，包含自訂 Docker image（26 個 CLI 工具）、RBAC、Cloudflare Tunnel、企業級測試套件（80+ 測試）。

---

## Cluster Info / 集群資訊

| Item | Value |
|------|-------|
| **K3s Version** | v1.34.3+k3s1 |
| **Nodes** | 2 master + 7 worker (amd64, Ubuntu 24.04) |
| **Namespace** | `hermes` |
| **Domain** | `hermes-woowtechmag.woowtech.io` |
| **LLM** | Minimax M2.7 (via `MINIMAX_API_KEY`) |
| **Storage** | local-path-provisioner |
| **Ingress** | Traefik (K3s default) |

---

## Architecture / 架構

```
Internet (HTTPS)
    |
    v
Cloudflare Edge (DDoS + TLS)
hermes-woowtechmag.woowtech.io
    |  QUIC Tunnel (khh01 + tpe01)
    v
+================================================+
| K3s Cluster — Namespace: hermes                |
|                                                |
|  cloudflared ──> hermes-webui :8787            |
|                      |                         |
|                      v                         |
|                 hermes-agent :8642 :9119        |
|                   |          |                  |
|                   v          v                  |
|              PostgreSQL   Redis                 |
|              :5432 10Gi   :6379 5Gi             |
+================================================+
```

---

## Directory Structure / 目錄結構

```
.
├── README.md                    # This file
├── Dockerfile.hermes-agent      # Custom image (26 CLI tools baked in)
├── build-image.sh               # Build + push script
├── deploy.sh                    # One-click deployment
├── init-cloudflare-hermes.py    # Cloudflare Tunnel auto-init
├── .env.example                 # Environment template
├── .gitignore
├── k8s-manifests/
│   ├── 00-namespace.yaml        # hermes namespace
│   ├── 01-secrets.yaml          # API keys, tokens (template)
│   ├── 01a-rbac.yaml            # ServiceAccount + RBAC
│   ├── 02-configmap.yaml        # Non-secret configuration
│   ├── 03-pvc.yaml              # PVC: PostgreSQL 10Gi, Redis 5Gi, Home 10Gi
│   ├── 04-postgresql.yaml       # PostgreSQL 15 Deployment + Service
│   ├── 05-redis.yaml            # Redis 7-alpine Deployment + Service
│   ├── 06-hermes-agent.yaml     # Hermes Agent (custom image) + Service
│   ├── 07-hermes-webui.yaml     # Hermes WebUI + initContainers + Service
│   ├── 08-cloudflared.yaml      # Cloudflare Tunnel Deployment
│   ├── 09-ingress.yaml          # Traefik Ingress (/ → WebUI, /api → Agent)
│   └── 10-network-policy.yaml   # NetworkPolicy (DB/Redis isolation)
└── tests/
    ├── config.env               # Test environment config
    ├── run-all.sh               # Master test runner (5 rounds + Playwright)
    ├── run-cli-tools-e2e.sh     # 15-scenario CLI tools E2E test
    ├── round1-infra.sh          # Infrastructure health (17 tests)
    ├── round2-api.sh            # Backend API (14 tests)
    ├── round3-security.sh       # Security & stress (16 tests)
    ├── round4-resilience.sh     # Resilience & recovery (11 tests)
    ├── round5-integration.sh    # Cross-service integration (10 tests)
    ├── lib/
    │   ├── assert.sh            # pass/fail/skip assertion library
    │   └── report.sh            # HTML report generator (dark theme)
    ├── playwright/
    │   ├── playwright.config.mjs
    │   └── hermes-webui.spec.mjs  # 12 browser E2E tests
    ├── PRD-hermes-enterprise-test.md
    └── TEST-REPORT-enterprise.md
```

---

## Deployment / 部署

### Prerequisites / 前置條件

- K3s cluster (v1.28+) with `kubectl` access
- Python 3 with `requests` module
- Cloudflare API token (Zone:Read, DNS:Edit, Tunnel:Edit)
- Minimax API key (https://www.minimax.io)

### Step 1: Initialize Cloudflare Tunnel

```bash
CF_API_TOKEN=<your-token> python3 init-cloudflare-hermes.py
```

This creates `cf-config.json` with tunnel ID, token, and account ID.

### Step 2: One-Click Deploy

```bash
MINIMAX_API_KEY=<your-key> ./deploy.sh
```

The deploy script will:
1. Create `hermes` namespace
2. Generate secrets (PostgreSQL password auto-generated)
3. Apply all manifests in order
4. Wait for databases → agent → WebUI to be ready
5. Configure Cloudflare Tunnel route
6. Show deployment status

### Step 3: Verify

```bash
kubectl get pods -n hermes
kubectl get svc -n hermes
```

Expected output: 5 pods all `1/1 Running`.

### Step 4: Access

Open `https://hermes-woowtechmag.woowtech.io` and login with the configured password.

---

## Custom Docker Image / 自訂映像

The agent uses a custom image built from `Dockerfile.hermes-agent` extending `nousresearch/hermes-agent:latest` (Debian 13 trixie).

### 26 CLI Tools Baked In

| Wave | Tools | Size |
|------|-------|------|
| **Wave 1**: Core | jq, yq (mikefarah), fd, rsync, git-lfs, mosh | ~30MB |
| **Wave 2**: Business | psql (17.9), redis-cli, kubectl (v1.34.3), helm (v3.17.3), argocd (v2.14.12), cloudflared, gh (2.73.0) | ~250MB |
| **Wave 3**: Content | pandoc, ImageMagick, gcloud SDK (gsutil/bq), httpie | ~350MB |
| **Wave 4**: Exploration | lynx, yt-dlp, nmap, dig, ping, nc, traceroute, chromium, playwright-cli | ~300MB |

### Build & Import

```bash
# Build
./build-image.sh

# Import to K3s
buildah push hermes-agent-custom:latest docker-archive:/tmp/hermes.tar
sudo k3s ctr images import /tmp/hermes.tar
```

---

## RBAC / 權限

`01a-rbac.yaml` creates:

| Resource | Name | Scope |
|----------|------|-------|
| **ServiceAccount** | `hermes-agent-sa` | hermes namespace |
| **ClusterRole** | `hermes-agent-cluster-reader` | Cluster-wide read-only |
| **ClusterRoleBinding** | `hermes-agent-cluster-reader-binding` | Binds SA to ClusterRole |
| **Role** | `hermes-agent-ns-writer` | hermes namespace write |
| **RoleBinding** | `hermes-agent-ns-writer-binding` | Binds SA to Role |

**Cluster-wide read**: pods, services, deployments, nodes, namespaces, events, PVC, ingress, network policies

**Namespace write** (hermes only): patch deployments, delete pods, view logs, exec, manage configmaps, view secrets

---

## Persistence / 持久化

| PVC | Size | Mount | Data |
|-----|------|-------|------|
| `hermes-home-pvc` | 10Gi | Agent `/opt/data` | config.yaml, .env, sessions, memories, skills, logs |
| `hermes-postgresql-pvc` | 10Gi | PostgreSQL `/var/lib/postgresql/data` | Database |
| `hermes-redis-pvc` | 5Gi | Redis `/data` | AOF persistence |

WebUI uses `emptyDir` volumes — config/tools rebuilt by initContainers on each restart.

---

## Network / 網路

### Services

| Service | Type | Ports | Selector |
|---------|------|-------|----------|
| `hermes-agent-svc` | ClusterIP | 8642 (gateway), 9119 (dashboard) | `app=hermes-agent` |
| `hermes-webui-svc` | ClusterIP | 8787 | `app=hermes-webui` |
| `hermes-postgresql-svc` | ClusterIP | 5432 | `app=hermes-postgresql` |
| `hermes-redis-svc` | ClusterIP | 6379 | `app=hermes-redis` |

### Ingress

| Path | Backend |
|------|---------|
| `/` | hermes-webui-svc:8787 |
| `/api` | hermes-agent-svc:8642 |

### Network Policies

- PostgreSQL: only `hermes-agent` and `hermes-webui` pods can connect on 5432
- Redis: only `hermes-agent` and `hermes-webui` pods can connect on 6379

---

## Testing / 測試

### Full Test Suite (80+ tests)

```bash
bash tests/run-all.sh
```

| Round | Tests | Pass Rate |
|-------|-------|-----------|
| R1 Infrastructure | 17 | 100% |
| R2 Backend API | 14 | 85%+ |
| R3 Security | 16 | 86%+ |
| R4 Resilience | 11 | 100% |
| R5 Integration | 10 | 100% |
| Playwright Browser | 12 | 100% |
| **Overall** | **80** | **96%+** |

### CLI Tools E2E (15 scenarios)

```bash
bash tests/run-cli-tools-e2e.sh
```

15 enterprise scenarios testing all 26 tools through real Hermes chat conversations via Playwright CLI:
- S1: kubectl cluster health
- S2: psql PostgreSQL query
- S3: redis-cli cache inspection
- S4: curl+jq JSON API processing
- S5: yq YAML config extraction
- S6: dig DNS + ping network
- S7: lynx web page retrieval
- S8: fd+rg file search
- S9: helm environment
- S10: gh GitHub CLI
- S11: argocd+cloudflared
- S12: ImageMagick create+identify
- S13: pandoc Markdown→HTML
- S14: nmap+nc port scan
- S15: git+rsync+traceroute multi-tool

---

## Troubleshooting / 故障排除

### Pod Not Starting

```bash
kubectl describe pod -n hermes -l app=hermes-agent
kubectl logs -n hermes deploy/hermes-agent --tail=30
```

### WebUI "AIAgent not available"

The WebUI needs hermes-agent source code. Check initContainer `clone-agent-src`:
```bash
kubectl logs -n hermes -l app=hermes-webui -c clone-agent-src
```

### Cloudflare Tunnel Not Connected

```bash
kubectl logs -n hermes deploy/cloudflared --tail=10
```
Look for `Registered tunnel connection` messages.

### CLI Tools Not Found in Chat

Tools are installed via WebUI's postStart hook (~60s). Check:
```bash
kubectl exec -n hermes deploy/hermes-webui -- cat /tmp/tools-install.log
```

---

## Management Commands / 管理指令

```bash
# View all resources
kubectl get all -n hermes

# View logs
kubectl logs -n hermes deploy/hermes-agent -f
kubectl logs -n hermes deploy/hermes-webui -f

# Restart a service
kubectl rollout restart deploy/hermes-agent -n hermes

# Enter PostgreSQL
kubectl exec -it -n hermes deploy/hermes-postgresql -- psql -U hermes -d hermes

# Enter Redis
kubectl exec -it -n hermes deploy/hermes-redis -- redis-cli

# Delete everything
kubectl delete namespace hermes
```

---

## License / 授權

MIT License — Copyright (c) 2026 Woowtech Smart Space Solution
