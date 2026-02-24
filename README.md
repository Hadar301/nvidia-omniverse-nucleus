# NVIDIA Omniverse Nucleus on Red Hat OpenShift

Production-ready deployment of NVIDIA Omniverse Nucleus on OpenShift using Docker-in-Docker (DIND).

## Why DIND?

NVIDIA Omniverse Nucleus is designed for Docker Compose. The DIND approach:
- ✅ Uses NVIDIA's official Docker Compose stack without modification
- ✅ Single LoadBalancer (cost-effective)
- ✅ Easy updates - drop in new NGC packages

See [DEPLOYMENT-SUMMARY.md](DEPLOYMENT-SUMMARY.md) for why native Kubernetes doesn't work.

## Quick Start

### Prerequisites

1. **OpenShift cluster** with 16+ cores, 32GB RAM, 500GB block storage
2. **NGC API key** from https://ngc.nvidia.com/
3. **NGC package** already included in `upstream/` directory

### Deploy

```bash
# 1. Configure NGC credentials
cat > .env <<EOF
NGC_API_KEY=your_ngc_api_key_here
NAMESPACE=omniverse-nucleus
EOF

# 2. Deploy
cd deploy-dind
./deploy-dind.sh

# 3. Get URL (wait 5-10 minutes for startup)
oc get svc nucleus-dind -n omniverse-nucleus -o jsonpath='{.status.loadBalancer.ingress[0].hostname}'
```

**Login**: Username `omniverse`, Password `omniverse123`

## Documentation

- [deploy-dind/README.md](deploy-dind/README.md) - Complete deployment guide with architecture and troubleshooting
- [DEPLOYMENT-SUMMARY.md](DEPLOYMENT-SUMMARY.md) - Why DIND vs native Kubernetes

### NVIDIA Official Docs

- [Nucleus Overview](https://docs.omniverse.nvidia.com/nucleus/)
- [Architecture Guide](https://docs.omniverse.nvidia.com/nucleus/latest/architecture.html)
- [Sizing Guide](https://docs.omniverse.nvidia.com/nucleus/latest/sizing-guide.html)
- [Enterprise Installation](https://docs.omniverse.nvidia.com/nucleus/latest/enterprise/installation/install-ove-nucleus.html)
- [NGC Catalog](https://catalog.ngc.nvidia.com/orgs/nvidia/teams/omniverse/resources/nucleus-compose-stack-pb24h2)

## Repository Structure

```
.
├── README.md                    # This file (quick start)
├── DEPLOYMENT-SUMMARY.md        # Why DIND vs native Kubernetes
├── .env                         # NGC credentials (create this)
├── upstream/                    # NGC packages (included)
│   └── nucleus-stack-*.tar.gz
└── deploy-dind/                 # DIND deployment
    ├── README.md                # Complete deployment guide
    ├── deploy-dind.sh           # Deploy script
    ├── cleanup-dind.sh          # Cleanup script
    ├── generate-secrets.sh      # Secret generation (auto-called)
    ├── nucleus-dind-simple.yaml # Pod definition
    └── privileged-sa.yaml       # ServiceAccount
```

## Common Operations

### Check Status
```bash
oc get pods -n omniverse-nucleus
oc logs -f deployment/nucleus-dind -n omniverse-nucleus
```

### Cleanup
```bash
cd deploy-dind
./cleanup-dind.sh  # Deletes all resources including data
```

### Update to Newer Version
1. Download new NGC package to `upstream/`
2. Run `./cleanup-dind.sh && ./deploy-dind.sh`

## Security (Change for Production!)

⚠️ Default deployment uses insecure sample secrets:
- Default passwords: `omniverse123`
- Sample cryptographic keys
- No SSL/TLS

For production: Update `.env` passwords, enable SSL/TLS, configure SSO. See [deploy-dind/README.md](deploy-dind/README.md).

## License

This deployment configuration is provided as-is. NVIDIA Omniverse Nucleus is proprietary software - see NVIDIA's licensing terms.
