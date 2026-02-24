# NVIDIA Omniverse Nucleus - Docker-in-Docker Deployment on OpenShift

This deployment runs NVIDIA Omniverse Nucleus using Docker-in-Docker (DIND) on Red Hat OpenShift, preserving the official Docker Compose architecture while adapting it for Kubernetes.

## Overview

NVIDIA Omniverse Nucleus is officially designed for Docker Compose on Ubuntu 22.04 LTS. This deployment strategy:

1. **Runs Docker-in-Docker**: Uses a privileged pod containing a Docker daemon
2. **Preserves Docker Compose**: Runs the official `nucleus-stack-no-ssl.yml` unchanged inside the pod
3. **Automatic LoadBalancer Configuration**: Detects AWS ELB hostname and configures services automatically
4. **NGINX Proxy**: Configures Navigator to proxy external API requests to internal Docker services

### Architecture

```
External Client (Browser)
    ↓
AWS LoadBalancer (ada857f8214ef4e1e856b39a164459a2-752788023.us-east-1.elb.amazonaws.com)
    ↓
OpenShift Pod (nucleus-dind)
    ├── Docker Daemon (DIND container)
    └── Docker Compose (12 services in Docker network)
        ├── nucleus-discovery:3333
        ├── nucleus-auth:3100
        ├── nucleus-api:3009
        ├── nucleus-navigator:80 (NGINX proxy)
        └── ... 8 more services
```

## Prerequisites

1. **OpenShift Cluster**:
   - At least 16 cores and 32GB RAM available
   - Block storage (AWS EBS gp3-csi recommended) - 500Gi minimum
   - Cluster admin access for privileged containers

2. **NGC Package**:
   - Download `nucleus-stack-2023.2.x+tag-xxx.tar.gz` from [NGC Catalog](https://catalog.ngc.nvidia.com/orgs/nvidia/teams/omniverse/resources/nucleus-compose-stack-pb24h2)
   - Place in repository root directory

3. **NGC API Key**:
   - Already configured in `.env` file
   - Used for pulling container images from `nvcr.io`

## Quick Start

### 1. Deploy Nucleus

```bash
./deploy-dind/deploy-dind.sh
```

This will:
- Create ServiceAccount with privileged SCC and RBAC permissions
- Extract NGC package and create ConfigMap with docker-compose files and crypto secrets
- Create 500Gi PVC for persistent data storage
- Deploy DIND pod with init container that waits for LoadBalancer
- Configure SERVER_IP_OR_HOST automatically
- Start all 12 Docker Compose services
- Configure NGINX proxy in Navigator

**Expected output:**
```
======================================
✅ DIND deployment initiated!
======================================

Monitor startup with:
  oc logs -f deployment/nucleus-dind -c nucleus-compose -n hacohen-omniverse

Get LoadBalancer URL:
  oc get svc nucleus-dind -n hacohen-omniverse -o jsonpath='{.status.loadBalancer.ingress[0].hostname}'
```

### 2. Monitor Startup

Docker Compose startup takes **5-10 minutes**. Monitor progress:

```bash
# Watch logs
oc logs -f deployment/nucleus-dind -c nucleus-compose -n hacohen-omniverse

# Check pod status
oc get pods -n hacohen-omniverse -l app=nucleus-dind

# Check all containers are healthy
oc exec deployment/nucleus-dind -c dind -n hacohen-omniverse -- docker ps
```

### 3. Get LoadBalancer URL

```bash
oc get svc nucleus-dind -n hacohen-omniverse -o jsonpath='{.status.loadBalancer.ingress[0].hostname}'
```

Example output: `a6baec0c9131b4b94a6f75d18c383d9a-752788023.us-east-1.elb.amazonaws.com`

### 4. Access Navigator

1. Open browser to: `http://<LoadBalancer-hostname>`
2. Login with:
   - **Username**: `omniverse`
   - **Password**: `omniverse123`

## Key Technical Details

### Networking Challenge

**Problem**: Docker Compose services need to:
- Communicate with each other using internal Docker network names (`nucleus-auth:3100`)
- Register with discovery service using a publicly accessible hostname
- Accept connections from external browsers through LoadBalancer

**Solution**:
1. **Init Container** (`wait-for-loadbalancer`):
   - Waits for LoadBalancer to provision (up to 5 minutes)
   - Queries LoadBalancer hostname using `oc` CLI with ServiceAccount RBAC permissions
   - Writes hostname to shared emptyDir volume: `/shared/loadbalancer-hostname`

2. **Main Container** (`nucleus-compose`):
   - Reads LoadBalancer hostname from shared volume
   - Updates `.env` file: `sed -i "/^SERVER_IP_OR_HOST=/c\SERVER_IP_OR_HOST=$LB_HOST" .env`
   - Starts Docker Compose with correct configuration
   - Configures NGINX in Navigator to proxy `/omni/*` requests to internal services

3. **Service Registration**:
   - Services register with discovery using LoadBalancer hostname
   - Example: `{"host": "a6baec0c9131b4b94a6f75d18c383d9a-752788023.us-east-1.elb.amazonaws.com", "port": 3100}`

### File Structure

```
nucleus-server/
├── deploy-dind/
│   ├── nucleus-dind-simple.yaml    # Main deployment with DIND + init container
│   └── privileged-sa.yaml          # ServiceAccount with RBAC for service queries
├── scripts/
│   ├── deploy-dind.sh              # Deployment script
│   └── validate-deployment.sh      # Validation tests
└── nucleus-stack-*.tar.gz          # NGC package (downloaded separately)
```

### Key Configuration Files

#### 1. Init Container (nucleus-dind-simple.yaml:36-57)

```yaml
initContainers:
- name: wait-for-loadbalancer
  image: registry.redhat.io/openshift4/ose-cli:latest
  command: ["/bin/bash", "-c"]
  args:
    - |
      echo "Waiting for LoadBalancer hostname..."
      for i in $(seq 1 60); do
        LB_HOST=$(oc get service nucleus-dind -n hacohen-omniverse -o jsonpath='{.status.loadBalancer.ingress[0].hostname}' 2>/dev/null || true)
        if [ -n "$LB_HOST" ]; then
          echo "LoadBalancer ready: $LB_HOST"
          echo "$LB_HOST" > /shared/loadbalancer-hostname
          exit 0
        fi
        echo "Waiting... ($i/60)"
        sleep 5
      done
  volumeMounts:
    - name: shared-data
      mountPath: /shared
```

#### 2. SERVER_IP_OR_HOST Configuration (nucleus-dind-simple.yaml:111-120)

```bash
# Set SERVER_IP_OR_HOST from LoadBalancer hostname (written by init container)
if [ -f /shared/loadbalancer-hostname ]; then
  LB_HOST=$(cat /shared/loadbalancer-hostname)
  echo "Setting SERVER_IP_OR_HOST to LoadBalancer: $LB_HOST"
  sed -i "/^SERVER_IP_OR_HOST=/c\SERVER_IP_OR_HOST=$LB_HOST" .env
  echo "✓ SERVER_IP_OR_HOST configured"
else
  echo "WARNING: LoadBalancer hostname file not found"
fi
```

#### 3. NGINX Proxy Configuration (nucleus-dind-simple.yaml:131-182)

```nginx
server {
    listen 80;
    server_name _;

    # Proxy /omni/discovery/ → nucleus-discovery:3333
    location /omni/discovery/ {
        proxy_pass http://nucleus-discovery:3333/;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection $connection_upgrade;
    }

    # Proxy /omni/auth/ → nucleus-auth:3100
    location /omni/auth/ {
        proxy_pass http://nucleus-auth:3100/;
    }

    # Proxy /omni/ → nucleus-api:3009
    location /omni/ {
        proxy_pass http://nucleus-api:3009/;
        proxy_read_timeout 86400;
    }

    # Static files for Navigator UI
    location / {
        root /usr/share/nginx/html;
        try_files $uri /index.html;
    }
}
```

## Services Exposed

The LoadBalancer Service exposes these ports:

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

- **PVC**: `nucleus-dind-data` (500Gi, gp3-csi)
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
oc logs deployment/nucleus-dind -c wait-for-loadbalancer -n hacohen-omniverse
```

If LoadBalancer isn't provisioning, check Service:
```bash
oc get svc nucleus-dind -n hacohen-omniverse -o yaml
```

### Services Not Starting

Check Docker daemon:
```bash
oc exec deployment/nucleus-dind -c dind -n hacohen-omniverse -- docker ps
```

Check docker-compose logs:
```bash
oc logs -f deployment/nucleus-dind -c nucleus-compose -n hacohen-omniverse
```

### Navigator Shows "Disconnected"

1. Check NGINX configuration inside Navigator container:
```bash
oc exec deployment/nucleus-dind -c dind -n hacohen-omniverse -- \
  docker exec base_stack-nucleus-navigator-1 cat /etc/nginx/sites-available/default
```

2. Verify services registered with LoadBalancer hostname:
```bash
oc exec deployment/nucleus-dind -c dind -n hacohen-omniverse -- \
  docker logs base_stack-nucleus-discovery-1 | grep "host"
```

Should show: `"host": "a6baec0c9131b4b94a6f75d18c383d9a-752788023.us-east-1.elb.amazonaws.com"`

### Permission Denied Errors

Verify privileged SCC is bound:
```bash
oc adm policy who-can use scc privileged -n hacohen-omniverse | grep nucleus-dind-sa
```

Should show: `nucleus-dind-sa`

## Cleanup

### Delete Deployment (Keep Data)

```bash
oc delete deployment nucleus-dind -n hacohen-omniverse
oc delete service nucleus-dind -n hacohen-omniverse
oc delete configmap nucleus-compose-files -n hacohen-omniverse
```

### Full Cleanup (Delete Everything)

```bash
oc delete deployment nucleus-dind -n hacohen-omniverse
oc delete service nucleus-dind -n hacohen-omniverse
oc delete pvc nucleus-dind-data -n hacohen-omniverse  # WARNING: Data loss!
oc delete configmap nucleus-compose-files -n hacohen-omniverse
oc delete secret crypto-secrets master-password service-password -n hacohen-omniverse
```

## Redeployment

To redeploy after cleanup:

```bash
./deploy-dind/deploy-dind.sh
```

All configuration is automated - no manual steps required!

## Advantages of DIND Approach

1. **Official Images**: Uses unmodified NVIDIA container images
2. **Docker Compose Compatibility**: Runs official `nucleus-stack-no-ssl.yml` without changes
3. **Simple Updates**: Download new NGC package and redeploy
4. **Familiar Troubleshooting**: Standard Docker Compose commands work inside pod
5. **Automatic Configuration**: LoadBalancer hostname detection and service configuration
6. **Persistent Storage**: Data survives pod restarts

## Limitations

1. **Privileged Mode Required**: DIND needs privileged security context
2. **Single Pod**: Cannot scale horizontally (Docker Compose limitation)
3. **Resource Overhead**: Docker-in-Docker adds some overhead vs native Kubernetes
4. **No SSL/TLS**: Current deployment uses non-SSL mode (can be upgraded)

## Next Steps

1. **Enable SSL/TLS**: Switch to `nucleus-stack-ssl.yml` with cert-manager
2. **SSO Integration**: Configure enterprise authentication (LDAP, OAuth)
3. **High Availability**: Implement automated backup/restore with Velero
4. **Monitoring**: Integrate Prometheus metrics (available on API port 3010)
5. **Production Secrets**: Replace sample crypto secrets with properly generated keys

## References

- [NVIDIA Omniverse Nucleus Documentation](https://docs.omniverse.nvidia.com/nucleus/latest/)
- [NGC Catalog - Nucleus Stack](https://catalog.ngc.nvidia.com/orgs/nvidia/teams/omniverse/resources/nucleus-compose-stack-pb24h2)
- [OpenShift Documentation](https://docs.openshift.com/)

## Support

For issues or questions:
1. Check logs: `oc logs -f deployment/nucleus-dind -c nucleus-compose -n hacohen-omniverse`
2. Verify pod status: `oc get pods -n hacohen-omniverse -l app=nucleus-dind`
3. Check Docker containers: `oc exec deployment/nucleus-dind -c dind -- docker ps`
