# NVIDIA Omniverse Nucleus - DIND Deployment Guide

Complete guide for deploying NVIDIA Omniverse Nucleus using Docker-in-Docker (DIND) on Red Hat OpenShift.

## Overview

This deployment runs NVIDIA's official Docker Compose stack inside an OpenShift pod using Docker-in-Docker. It automatically configures the LoadBalancer hostname and NGINX proxy for external access.

### Architecture

```
External Browser
    ↓
AWS LoadBalancer (ELB)
    ↓
OpenShift Pod (nucleus-dind)
    ├── Docker Daemon (DIND container)
    └── Docker Compose (12 services in Docker network)
        ├── nucleus-navigator:80 (NGINX proxy)
        ├── nucleus-discovery:3333
        ├── nucleus-auth:3100
        ├── nucleus-api:3009
        └── ... 8 more services
```

## Prerequisites

### 1. OpenShift Cluster

- **Resources**: 16+ cores, 32GB RAM minimum (32+ cores, 64GB for production)
- **Storage**: 500GB block storage (AWS EBS gp3-csi recommended)
- **LoadBalancer**: AWS ELB support
- **Permissions**: Cluster admin access for privileged containers

### 2. NGC Package

**Already included** in `upstream/nucleus-stack-2023.2.9+tag-xxx.tar.gz`

To download a newer version:

1. Visit [NGC Catalog - Nucleus Stack](https://catalog.ngc.nvidia.com/orgs/nvidia/teams/omniverse/resources/nucleus-compose-stack/files?version=2023.2.9)
2. Sign in with your NVIDIA account (free)
3. Navigate to **Files** tab
4. Select the latest version from dropdown
5. Click **Download** next to `nucleus-stack-2023.2.x+tag-xxx.tar.gz`
6. Save to `upstream/` directory

```bash
mv ~/Downloads/nucleus-stack-*.tar.gz upstream/
```

### 3. NGC API Key

Get your API key:
1. Go to [NGC Account Settings](https://ngc.nvidia.com/setup)
2. Click **Generate API Key**
3. Copy and save to `.env` file:

```bash
cat > .env <<EOF
NGC_API_KEY=your_ngc_api_key_here
NAMESPACE=omniverse-nucleus
EOF
```

## Deploy

```bash
./deploy-dind/deploy-dind.sh
```

The script automatically:
- Creates ServiceAccount with privileged SCC
- Generates crypto secrets if they don't exist
- Extracts NGC package and creates ConfigMap
- Creates 500Gi PVC for persistent data
- Deploys DIND pod with init container that waits for LoadBalancer
- Configures `SERVER_IP_OR_HOST` automatically
- Starts all 12 Docker Compose services
- Configures NGINX proxy in Navigator

**Wait time**: 5-10 minutes for Docker Compose to start all services

## Monitor Startup

```bash
# Watch logs
oc logs -f deployment/nucleus-dind -c nucleus-compose -n omniverse-nucleus

# Check pod status
oc get pods -n omniverse-nucleus -l app=nucleus-dind

# Check Docker containers are healthy
oc exec deployment/nucleus-dind -c dind -n omniverse-nucleus -- docker ps
```

## Access Navigator

```bash
# Get LoadBalancer URL
oc get svc nucleus-dind -n omniverse-nucleus -o jsonpath='{.status.loadBalancer.ingress[0].hostname}'
```

Open browser to: `http://<loadbalancer-hostname>`

**Login**:
- Username: `omniverse`
- Password: `omniverse123`

## How It Works

### Networking Challenge

Docker Compose services need to:
- Communicate internally using Docker network names (`nucleus-auth:3100`)
- Register with discovery using a public hostname
- Accept external browser connections through LoadBalancer

### Solution: Init Container + NGINX Proxy

**1. Init Container** (`wait-for-loadbalancer`):
- Waits up to 5 minutes for LoadBalancer to provision
- Queries hostname using `oc` CLI with ServiceAccount RBAC permissions
- Writes hostname to shared volume: `/shared/loadbalancer-hostname`

**2. Main Container** (`nucleus-compose`):
- Reads LoadBalancer hostname from shared volume
- Updates `.env` file: `sed -i "/^SERVER_IP_OR_HOST=/c\SERVER_IP_OR_HOST=$LB_HOST" .env`
- Starts Docker Compose with correct configuration
- Configures NGINX in Navigator to proxy `/omni/*` requests to internal services

**3. Service Registration**:
Services register with discovery using LoadBalancer hostname:
```json
{"host": "a6baec0c9131b4b94a6f75d18c383d9a-752788023.us-east-1.elb.amazonaws.com", "port": 3100}
```

## Services Exposed

| Service | Port | Protocol | Purpose |
|---------|------|----------|---------|
| Navigator | 80 | HTTP | Web UI + NGINX proxy |
| Discovery | 3333 | WebSocket | Service discovery |
| Auth | 3100 | WebSocket | Authentication |
| Auth SSL | 3180 | WebSocket | Auth (SSL mode) |
| Auth Admin | 8000 | HTTP | Auth admin API |
| API | 3009 | WebSocket | Core Nucleus API |
| API Admin | 3106 | HTTP | API admin |
| LFT | 3030 | HTTP | Large File Transfer |
| Search | 3400 | HTTP | File search/indexing |
| Tagging | 3020 | HTTP | File tagging |

## Persistent Storage

- **PVC**: `nucleus-dind-data` (500Gi)
- **Mount**: `/var/lib/omni/nucleus-data` inside Docker containers
- **Contains**:
  - Core database: `/var/lib/omni/nucleus-data/data`
  - Auth database: `/var/lib/omni/nucleus-data/local-accounts-db/`
  - Tags database: `/var/lib/omni/nucleus-data/tags-db/`
  - Logs: `/var/lib/omni/nucleus-data/log/`

## Troubleshooting

### Pod Stuck in Init:0/1

Check init container logs:
```bash
oc logs deployment/nucleus-dind -c wait-for-loadbalancer -n omniverse-nucleus
```

If LoadBalancer isn't provisioning:
```bash
oc get svc nucleus-dind -n omniverse-nucleus -o yaml
```

### Services Not Starting

Check Docker daemon:
```bash
oc exec deployment/nucleus-dind -c dind -n omniverse-nucleus -- docker ps
```

Check docker-compose logs:
```bash
oc logs -f deployment/nucleus-dind -c nucleus-compose -n omniverse-nucleus
```

### Navigator Shows "Disconnected"

1. Check NGINX configuration:
```bash
oc exec deployment/nucleus-dind -c dind -n omniverse-nucleus -- \
  docker exec base_stack-nucleus-navigator-1 cat /etc/nginx/sites-available/default
```

2. Verify services registered with LoadBalancer hostname:
```bash
oc exec deployment/nucleus-dind -c dind -n omniverse-nucleus -- \
  docker logs base_stack-nucleus-discovery-1 | grep "host"
```

Should show LoadBalancer hostname, not internal service names.

### Permission Denied Errors

Verify privileged SCC is bound:
```bash
oc adm policy who-can use scc privileged -n omniverse-nucleus | grep nucleus-dind-sa
```

Should show: `nucleus-dind-sa`

## Cleanup

```bash
./deploy-dind/cleanup-dind.sh
```

Prompts for confirmation before deleting:
- Deployment and Service
- PVC (⚠️ data loss!)
- ConfigMaps
- ServiceAccount and RBAC
- Secrets

## Files

- `deploy-dind.sh` - Main deployment script
- `cleanup-dind.sh` - Cleanup script
- `nucleus-dind-simple.yaml` - Pod and LoadBalancer definitions
- `privileged-sa.yaml` - ServiceAccount with RBAC permissions
- `generate-secrets.sh` - Crypto secret generation (auto-called)

## NVIDIA Documentation

- [Nucleus Overview](https://docs.omniverse.nvidia.com/nucleus/)
- [Architecture Guide](https://docs.omniverse.nvidia.com/nucleus/latest/architecture.html)
- [Sizing Guide](https://docs.omniverse.nvidia.com/nucleus/latest/sizing-guide.html)
- [Enterprise Installation](https://docs.omniverse.nvidia.com/nucleus/latest/enterprise/installation/install-ove-nucleus.html)
- [NGC Catalog](https://catalog.ngc.nvidia.com/orgs/nvidia/teams/omniverse/resources/nucleus-compose-stack-pb24h2)
