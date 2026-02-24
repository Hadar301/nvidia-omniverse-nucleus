# Why DIND Instead of Native Kubernetes?

After extensive testing, **DIND is the only working solution** for deploying NVIDIA Omniverse Nucleus on OpenShift.

## The Problem with Native Kubernetes

We attempted native Kubernetes deployment with individual Deployments, LoadBalancers, and NGINX proxy. The backend infrastructure worked perfectly:
- ✅ All services registered with LoadBalancer hostnames
- ✅ NGINX WebSocket proxying tested successfully
- ✅ Service-to-service communication worked

**But the Navigator web UI failed** because:
- ❌ JavaScript has hardcoded fallback to internal service names (`nucleus-api:3333`)
- ❌ These internal DNS names don't resolve from browsers
- ❌ `settings.json` configuration exists but JavaScript doesn't use it properly

## Why DIND Works

DIND solves this by running all services in a Docker network inside the pod:
- Internal hostnames like `nucleus-api` resolve via Docker DNS
- NGINX proxy handles external browser connections
- Uses NVIDIA's official Docker Compose without modifications
- Single LoadBalancer (cost-effective)

## Conclusion

Use the DIND deployment in [deploy-dind/](deploy-dind/) - it's the production-ready solution.
