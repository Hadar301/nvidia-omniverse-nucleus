# NVIDIA Omniverse Nucleus on Red Hat OpenShift

Production-ready deployment of NVIDIA Omniverse Nucleus on Red Hat OpenShift using Docker-in-Docker (DIND).

## Overview

This repository contains a proven deployment approach for running NVIDIA Omniverse Nucleus on OpenShift. After extensive testing of both native Kubernetes and DIND approaches, **the DIND deployment is the recommended production solution**.

### Why DIND?

NVIDIA Omniverse Nucleus is designed exclusively for Docker Compose deployments. The DIND approach:
- ✅ **Works immediately** - Uses NVIDIA's official Docker Compose stack without modification
- ✅ **Proven stable** - Leverages NVIDIA's tested deployment method
- ✅ **Complete feature support** - All services work exactly as designed
- ✅ **Single LoadBalancer** - Cost-effective AWS ELB usage
- ✅ **Easy updates** - Drop in new NGC packages when NVIDIA releases updates

## Quick Start

### Prerequisites

1. **Red Hat OpenShift cluster** with:
   - At least 16 CPU cores and 32GB RAM available
   - Block storage (AWS EBS, Ceph RBD, etc.) - minimum 500GB
   - LoadBalancer support (AWS ELB, etc.)

2. **NGC API Key** from NVIDIA:
   - Sign up at https://ngc.nvidia.com/
   - Generate API key from Account Settings
   - Save to `.env` file (see below)

3. **OpenShift CLI** (`oc`) installed and authenticated

### Installation

1. **Clone this repository**:
```bash
git clone https://github.com/your-org/nvidia-omniverse-nucleus.git
cd nvidia-omniverse-nucleus
```

2. **Configure NGC credentials**:
```bash
# Edit .env file and add your NGC API key
cat > .env <<EOF
NGC_API_KEY=your_ngc_api_key_here
NAMESPACE=omniverse-nucleus
EOF
```

3. **Deploy to OpenShift**:
```bash
cd deploy-dind
./deploy-dind.sh
```

The script will:
- Create namespace and service account
- Generate cryptographic secrets
- Extract NGC package
- Deploy DIND pod with all Nucleus services
- Create LoadBalancer for external access

4. **Access Navigator Web UI**:
```bash
# Get LoadBalancer URL
oc get service nucleus-dind -n omniverse-nucleus

# Open in browser:
# http://<loadbalancer-hostname>

# Login credentials:
# Username: omniverse
# Password: omniverse123
```

## Architecture

```
┌─────────────────────────────────────────────────────┐
│  AWS LoadBalancer (ELB)                             │
│  http://abc123.us-east-1.elb.amazonaws.com          │
└──────────────────┬──────────────────────────────────┘
                   │
                   ▼
┌─────────────────────────────────────────────────────┐
│  OpenShift Pod: nucleus-dind                        │
│  ┌───────────────────────────────────────────────┐  │
│  │  Docker-in-Docker Container                   │  │
│  │  ┌─────────────────────────────────────────┐  │  │
│  │  │  NVIDIA Nucleus Services (docker-compose)│  │  │
│  │  │  • Navigator (Web UI) - port 80         │  │  │
│  │  │  • Discovery - port 3333                │  │  │
│  │  │  • Auth - ports 3100, 3180, 8000        │  │  │
│  │  │  • Core API - ports 3009, 3106          │  │  │
│  │  │  • LFT (file transfer) - port 3030      │  │  │
│  │  │  • Search, Tagging, Thumbnails          │  │  │
│  │  └─────────────────────────────────────────┘  │  │
│  └───────────────────────────────────────────────┘  │
│         │                                            │
│         ▼                                            │
│  Persistent Volume (500GB)                           │
│  /var/lib/omni/nucleus-data                          │
└─────────────────────────────────────────────────────┘
```

### Key Components

1. **DIND Pod**: Runs Docker daemon inside OpenShift pod
2. **Docker Compose**: Orchestrates NVIDIA's official microservices
3. **LoadBalancer**: AWS ELB exposes all services on different ports
4. **Persistent Storage**: 500GB block storage for Nucleus database
5. **NGINX Proxy**: Routes browser requests to internal services

## Repository Structure

```
.
├── README.md                    # This file
├── .env                         # NGC credentials (not committed)
├── upstream/                    # NVIDIA NGC packages
│   └── nucleus-stack-*.tar.gz
└── deploy-dind/                 # DIND deployment files
    ├── README.md                # Detailed DIND documentation
    ├── QUICKSTART.md            # Quick reference
    ├── deploy-dind.sh           # Main deployment script
    ├── cleanup-dind.sh          # Cleanup script
    ├── nucleus-dind-simple.yaml # Pod and service definition
    └── privileged-sa.yaml       # Service account for DIND
```

## Usage

### Deploy

```bash
cd deploy-dind
./deploy-dind.sh
```

### Check Status

```bash
# Pod status
oc get pods -n omniverse-nucleus

# Service URLs
oc get svc nucleus-dind -n omniverse-nucleus

# Logs
oc logs -f deployment/nucleus-dind -n omniverse-nucleus
```

### Cleanup

```bash
cd deploy-dind
./cleanup-dind.sh
```

## Configuration

### Resource Requirements

**Minimum (POC)**:
- CPU: 16 cores
- Memory: 32GB
- Storage: 500GB

**Recommended (Production)**:
- CPU: 32+ cores
- Memory: 64GB
- Storage: 1TB+

### Environment Variables

Edit `.env` file:

```bash
# NGC credentials
NGC_API_KEY=your_api_key_here

# OpenShift namespace
NAMESPACE=omniverse-nucleus

# Admin credentials (change for production!)
MASTER_PASSWORD=omniverse123
SERVICE_PASSWORD=omniverse123
```

## Networking

The LoadBalancer exposes these ports:

| Port | Service | Protocol | Purpose |
|------|---------|----------|---------|
| 80 | Navigator | HTTP | Web UI |
| 3333 | Discovery | WebSocket | Service registry |
| 3100 | Auth | HTTP | Authentication API |
| 3180 | Auth | HTTP | Login form |
| 8000 | Auth | HTTP | Admin API |
| 3009 | Core API | HTTP | Main API |
| 3030 | LFT | HTTP | Large file transfer |

## Security

### For POC/Development

The deployment uses sample insecure secrets from NVIDIA's reference implementation:
- ⚠️ Default passwords: `omniverse123`
- ⚠️ Sample cryptographic keys
- ⚠️ No SSL/TLS

### For Production

Before production use:

1. **Generate proper secrets**:
```bash
# The deploy script generates sample secrets
# Replace these with production-grade keys
```

2. **Enable SSL/TLS**:
   - Switch to `nucleus-stack-ssl.yml` in the DIND pod
   - Configure certificates

3. **Change passwords**:
   - Update `MASTER_PASSWORD` and `SERVICE_PASSWORD` in `.env`
   - Redeploy

4. **Integrate SSO**:
   - Configure SAML or OAuth in Auth service
   - See NVIDIA Nucleus documentation

## Troubleshooting

### Pod won't start

```bash
# Check events
oc describe pod -l app=nucleus-dind -n omniverse-nucleus

# Common issues:
# - NGC credentials incorrect
# - Insufficient resources
# - Storage class not available
```

### Navigator shows "disconnected"

```bash
# Check if services are running inside DIND
oc exec deployment/nucleus-dind -n omniverse-nucleus -- docker ps

# All services should show "Up"
```

### LoadBalancer not provisioning

```bash
# AWS ELB can take 2-5 minutes
# Check status:
oc get svc nucleus-dind -n omniverse-nucleus

# Look for EXTERNAL-IP
```

### Storage issues

```bash
# Verify PVC is bound
oc get pvc nucleus-data -n omniverse-nucleus

# Should show "Bound" status
# If Pending, check StorageClass availability
```

## Documentation

- [deploy-dind/README.md](deploy-dind/README.md) - Detailed DIND deployment guide
- [deploy-dind/QUICKSTART.md](deploy-dind/QUICKSTART.md) - Quick reference
- [NVIDIA Nucleus Docs](https://docs.omniverse.nvidia.com/nucleus/) - Official documentation

## Alternatives Considered

### Native Kubernetes Deployment (Not Recommended)

We extensively tested a native Kubernetes deployment approach using:
- Individual service Deployments/StatefulSets
- Kubernetes Services and LoadBalancers
- ConfigMaps for configuration
- NGINX reverse proxy for browser access

**Result**: The Navigator web UI's JavaScript has hardcoded assumptions about direct service connectivity that don't work with browser-based access through proxies. While all backend services work correctly, the frontend cannot connect properly without extensive JavaScript modifications.

**Recommendation**: Use DIND deployment instead.

## Updates

To update to a newer NVIDIA Nucleus version:

1. Download new NGC package from https://catalog.ngc.nvidia.com/
2. Place in `upstream/` directory
3. Update `deploy-dind.sh` if package name changed
4. Run cleanup and redeploy:
```bash
./cleanup-dind.sh
./deploy-dind.sh
```

## Support

- **NVIDIA Nucleus**: https://docs.omniverse.nvidia.com/nucleus/
- **NGC Support**: https://ngc.nvidia.com/support
- **OpenShift**: https://access.redhat.com/support

## License

This deployment configuration is provided as-is. NVIDIA Omniverse Nucleus is proprietary software - see NVIDIA's licensing terms.

## Contributing

Contributions welcome! Please:
1. Fork the repository
2. Create a feature branch
3. Test thoroughly on OpenShift
4. Submit pull request

## Acknowledgments

- NVIDIA for Omniverse Nucleus
- Red Hat for OpenShift platform
- The community for feedback and testing
