# Woow Hermes Agent — AI Smart Home Assistant Deployment

**Hermes AI Agent — Multi-Environment Deployment**
**Hermes AI 智慧助理 — 多環境部署方案**

Deploy Hermes AI smart home assistant with WebUI, Minimax M2.7 LLM, 26 CLI tools, PostgreSQL, Redis, Cloudflare Tunnel, and enterprise-grade test suite.

部署 Hermes AI 智慧家庭助理，包含 WebUI、Minimax M2.7 大語言模型、26 個 CLI 工具、PostgreSQL、Redis、Cloudflare Tunnel、企業級測試套件。

---

## Deployment Options / 部署方式

| Branch | Environment | Description |
|--------|-------------|-------------|
| **`k3s`** | K3s Cluster | Production multi-node deployment with K8s manifests, custom Docker image, RBAC, Cloudflare Tunnel |
| **`main`** | Reference | Architecture docs, shared manifests, deployment guides |
| **`podman`** | Podman | Single-node deployment with podman-compose (coming soon) |

> **For K3s installation, switch to the [`k3s`](../../tree/k3s) branch.**

---

## Architecture / 架構

```
Internet (HTTPS)
    |
    v
+------------------------------------------------+
|  Cloudflare Edge (DDoS + TLS)                  |
|  hermes-woowtechmag.woowtech.io               |
+------------------------------------------------+
    |  QUIC Tunnel (khh01 + tpe01)
    v
+================================================+
| K3s Cluster — Namespace: hermes                |
|  (2 master + 7 worker nodes, amd64)            |
|                                                |
|  +----------------+   +--------------------+   |
|  | cloudflared    |-->| hermes-webui       |   |
|  | (tunnel)       |   | :8787              |   |
|  +----------------+   | +-- Chat UI        |   |
|                       | +-- Sessions       |   |
|                       | +-- Skills         |   |
|                       | +-- Memory         |   |
|                       | +-- 26 CLI Tools   |   |
|                       +--------+-----------+   |
|                                |               |
|                       +--------------------+   |
|                       | hermes-agent       |   |
|                       | :8642 (gateway)    |   |
|                       | :9119 (dashboard)  |   |
|                       | +-- Minimax M2.7   |   |
|                       | +-- API Server     |   |
|                       | +-- Cron/Tasks     |   |
|                       | +-- 87 Skills      |   |
|                       +--------+-----------+   |
|                                |               |
|                    +-----------+-----------+   |
|                    |                       |   |
|              +-----------+         +-------+   |
|              | PostgreSQL |         | Redis |   |
|              | :5432      |         | :6379 |   |
|              | 10Gi PVC   |         | 5Gi   |   |
|              +-----------+         +-------+   |
+================================================+
```

---

## Services / 服務

| Service | Image | Port | Description |
|---------|-------|------|-------------|
| **Hermes Agent** | `hermes-agent-custom:latest` | 8642, 9119 | AI gateway (Minimax M2.7), 87 skills, cron, API server |
| **Hermes WebUI** | `ghcr.io/nesquena/hermes-webui:latest` | 8787 | Chat UI, sessions, memory, workspace, 26 CLI tools |
| **PostgreSQL** | `postgres:15` | 5432 | Database (10Gi PVC, pg_isready health check) |
| **Redis** | `redis:7-alpine` | 6379 | Cache + task queue (5Gi PVC, AOF persistence) |
| **Cloudflared** | `cloudflare/cloudflared:latest` | 20241 | Cloudflare Tunnel (QUIC, multi-region) |

---

## CLI Tools (26) / 命令列工具

Custom Docker image includes 26 enterprise CLI tools baked in:

### Wave 1: Core Productivity
`jq` `yq` `fd` `rsync` `git-lfs` `mosh`

### Wave 2: Business Integration
`psql` `redis-cli` `kubectl` `helm` `argocd` `cloudflared` `gh`

### Wave 3: Content Generation
`pandoc` `ImageMagick` `gcloud` `httpie`

### Wave 4: Exploration
`lynx` `nmap` `dig` `ping` `nc` `traceroute` `chromium` `playwright-cli` `yt-dlp`

---

## K3s Quick Start / K3s 快速開始

```bash
# 1. Clone and switch to k3s branch
git clone https://github.com/WOOWTECH/Woow_hermes_agent_docker_compose_all.git
cd Woow_hermes_agent_docker_compose_all
git checkout k3s

# 2. Initialize Cloudflare Tunnel
CF_API_TOKEN=<your-token> python3 init-cloudflare-hermes.py

# 3. Deploy
./deploy.sh

# 4. Verify
kubectl get pods -n hermes
```

See [`k8s-manifests/`](k8s-manifests/) for full manifest reference.

---

## K3s Manifests / K3s 清單

| Manifest | Component |
|----------|-----------|
| `00-namespace.yaml` | Namespace `hermes` |
| `01-secrets.yaml` | API keys, tokens, credentials |
| `01a-rbac.yaml` | ServiceAccount + ClusterRole (read) + Role (write) |
| `02-configmap.yaml` | Cloudflare config, Hermes settings |
| `03-pvc.yaml` | PVC: PostgreSQL 10Gi, Redis 5Gi, Hermes Home 10Gi |
| `04-postgresql.yaml` | PostgreSQL 15 + Service |
| `05-redis.yaml` | Redis 7-alpine + AOF + Service |
| `06-hermes-agent.yaml` | Hermes Agent (custom image) + Service |
| `07-hermes-webui.yaml` | Hermes WebUI + initContainers + CLI tools |
| `08-cloudflared.yaml` | Cloudflare Tunnel connector |
| `09-ingress.yaml` | Traefik Ingress (/ → WebUI, /api → Agent) |
| `10-network-policy.yaml` | DB/Redis access restricted to app pods |

---

## Testing / 測試

Enterprise-grade test suite with 80+ tests:

| Round | Tests | Coverage |
|-------|-------|----------|
| Round 1 | 17 | Infrastructure health (pods, services, PVC, network policies) |
| Round 2 | 14 | Backend API (PostgreSQL CRUD, Redis, DNS, HTTPS) |
| Round 3 | 16 | Security & stress (XSS, SQLi, concurrent, TLS) |
| Round 4 | 11 | Resilience (pod kill, data persistence, scaling, rollback) |
| Round 5 | 10 | Cross-service integration (ingress routing, ConfigMap, tunnel) |
| Playwright | 12 | Browser E2E (auth, responsive, navigation, performance) |
| CLI E2E | 15 | CLI tools verification via real chat conversations |

```bash
# Run full test suite
bash tests/run-all.sh

# Run CLI tools E2E
bash tests/run-cli-tools-e2e.sh
```

---

## Security / 安全

- **RBAC**: ServiceAccount with cluster-wide read + namespace-scoped write
- **Network Policies**: PostgreSQL and Redis restricted to hermes-agent and hermes-webui pods
- **Secrets**: API keys stored in K8s Secrets (base64), never in ConfigMaps
- **Password Auth**: WebUI protected with password authentication
- **TLS**: Cloudflare Tunnel provides end-to-end HTTPS
- **Non-root**: Agent runs as hermes user (UID 10000)

---

## License / 授權

MIT License — Copyright (c) 2026 Woowtech Smart Space Solution
