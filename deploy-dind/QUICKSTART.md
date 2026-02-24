# DIND Deployment - Quick Reference

## Prerequisites

1. NGC package in repository root: `nucleus-stack-*.tar.gz`
2. Secrets created: `make secrets` (from repository root)
3. Cluster admin access for privileged containers

## Deploy

```bash
./deploy-dind/deploy-dind.sh
```

**Wait time**: 5-10 minutes for Docker Compose to start all services

## Access

```bash
# Get LoadBalancer URL
oc get svc nucleus-dind -n hacohen-omniverse -o jsonpath='{.status.loadBalancer.ingress[0].hostname}'
```

**Login**:
- Username: `omniverse`
- Password: `omniverse123`

## Monitor

```bash
# Watch startup logs
oc logs -f deployment/nucleus-dind -c nucleus-compose -n hacohen-omniverse

# Check pod status
oc get pods -n hacohen-omniverse -l app=nucleus-dind

# Check Docker containers
oc exec deployment/nucleus-dind -c dind -n hacohen-omniverse -- docker ps
```

## Cleanup

```bash
./deploy-dind/cleanup-dind.sh
```

This will prompt for confirmation before deleting resources.

## Files

- `README.md` - Complete documentation with architecture and troubleshooting
- `deploy-dind.sh` - Deployment script
- `cleanup-dind.sh` - Cleanup script
- `nucleus-dind-simple.yaml` - Main deployment manifest with init container
- `privileged-sa.yaml` - ServiceAccount with RBAC permissions

## Networking Solution

**Problem**: Docker Compose services need LoadBalancer hostname, but it's not available until after deployment.

**Solution**:
1. Init container waits for LoadBalancer to provision
2. Queries hostname using `oc` CLI with ServiceAccount permissions
3. Writes hostname to shared volume
4. Main container reads hostname and configures `SERVER_IP_OR_HOST` in `.env`
5. Services register with discovery using LoadBalancer hostname
6. NGINX in Navigator proxies `/omni/*` to internal Docker services

## Support

See [README.md](README.md) for complete documentation.
