# Homelab Flux Controller

A GitOps-based Kubernetes deployment setup using Flux CD for managing homelab applications with proper infrastructure separation.

> **Cluster engine:** [k3s](https://k3s.io/) (lightweight Kubernetes) running on a Raspberry Pi (arm64).
> Installed with Traefik disabled — external routing is handled entirely by a Cloudflare Tunnel
> (`cloudflared`) to in-cluster ClusterIP services. Persistent storage uses the built-in
> `local-path` StorageClass.
>
> The Kubernetes version is pinned to the **v1.35** channel because the current stable Rancher
> chart requires `kubeVersion < 1.36`. Reinstall/upgrade with:
> `curl -sfL https://get.k3s.io | sudo INSTALL_K3S_CHANNEL=v1.35 INSTALL_K3S_EXEC="--disable=traefik --write-kubeconfig-mode=644" sh -`

## 🏗️ Project Structure

This repository follows GitOps best practices with clear separation between infrastructure, image automation, and application workloads.

```
homelab-flux-controller/
├── README.md                                    # This file
├── infrastructure/                              # Infrastructure components
│   ├── kustomization.yaml                      # Infrastructure orchestration
│   └── networking/                              # Networking infrastructure
│       ├── kustomization.yaml
│       └── ingress/                             # Centralized ingress routing
│           ├── ingress.yaml                     # Multi-service ingress configuration
│           └── kustomization.yaml               # Ingress resource management
├── apps/                                        # Application definitions (WHAT to deploy)
│   ├── kustomization.yaml                      # Main apps orchestration
│   └── nginx/                                   # Nginx web server application
│       ├── deployment.yaml                     # Kubernetes Deployment manifest
│       ├── service.yaml                        # Kubernetes Service manifest
│       └── kustomization.yaml                  # App resource organization
└── clusters/                                    # Cluster configurations (HOW to deploy)
    └── homelab/                                 # Homelab cluster
        ├── kustomization.yaml                   # Cluster orchestration
        ├── flux-system/                         # Flux CD core components
        │   ├── gotk-components.yaml             # Flux controllers
        │   ├── gotk-sync.yaml                   # Git sync configuration
        │   └── kustomization.yaml
        └── apps/                                # Cluster deployment configurations
            ├── kustomization.yaml               # Deployment orchestration
            ├── infrastructure.yaml              # Infrastructure deployment
            ├── image-automation.yaml            # Image automation deployment
            ├── apps.yaml                        # Applications deployment
            ├── apps/                            # Individual app deployments
            │   └── nginx.yaml                   # Nginx deployment config
            └── image-automation/                # Image automation configurations
                ├── kustomization.yaml
                └── nginx/                       # Nginx image automation
                    ├── kustomization.yaml
                    ├── nginx-repository.yaml    # Image repository to watch
                    ├── nginx-image-policy.yaml  # Image update policy
                    └── nginx-image-update.yaml  # Image update automation
```

## 🎯 Design Principles

### **Infrastructure Separation**
- **`infrastructure/`**: Networking, ingress, monitoring, security components
- **`apps/`**: Application workloads and business logic
- **`clusters/`**: Deployment orchestration and cluster-specific configurations

### **Deployment Hierarchy**
```
1. Infrastructure → 2. Image Automation → 3. Applications
```

### **GitOps Flow**
```
Git Repository → Flux System → Infrastructure → Image Automation → Applications → Kubernetes Resources
```

## 🚀 How It Works

### **1. Flux Bootstrap**
Flux monitors this Git repository and automatically applies changes to your Kubernetes cluster.

**Entry Point**: `clusters/homelab/flux-system/gotk-sync.yaml`
```yaml
spec:
  path: ./clusters/homelab  # Flux watches this path
```

### **2. Cluster Configuration**
The cluster kustomization orchestrates deployment order:

**File**: `clusters/homelab/apps/kustomization.yaml`
```yaml
resources:
  - infrastructure.yaml     # Deploy infrastructure first
  - image-automation.yaml   # Deploy image automation second
  - apps.yaml              # Deploy applications last
```

### **3. Infrastructure Deployment**
Infrastructure components are deployed first:

**File**: `clusters/homelab/apps/infrastructure.yaml`
```yaml
spec:
  path: ./infrastructure    # Points to infrastructure definitions
  interval: 10m0s           # Longer interval for stability
```

### **4. Image Automation Deployment**
Image automation components are deployed in flux-system namespace:

**File**: `clusters/homelab/apps/image-automation.yaml`
```yaml
spec:
  path: ./clusters/homelab/apps/image-automation
  interval: 30m             # Image automation runs less frequently
```

### **5. Application Deployment**
Applications depend on infrastructure and image automation:

**File**: `clusters/homelab/apps/apps.yaml`
```yaml
spec:
  path: ./apps              # Points to application definitions
  dependsOn:
    - name: infrastructure
    - name: image-automation
```

### **6. Centralized Ingress Routing (Cloudflare Tunnel)**
Ingress in this repository is managed by a GitOps-controlled Cloudflare Tunnel workload (no in-cluster ingress controller / Traefik is used). The tunnel maps external hostnames to in-cluster services using internal DNS (ClusterIP). This removes the need to expose NodePorts on the cluster nodes.

Manifests:

- `infrastructure/networking/cloudflared/kustomization.yaml` — Kustomization for the Cloudflare Tunnel
- `infrastructure/networking/cloudflared/deployment.yaml` — `cloudflared` Deployment
- `infrastructure/networking/cloudflared/configmap-tunnel.yaml` — Tunnel ingress rules / hostnames

Notes:

- The ConfigMap contains `tunnel-config.yml` which is the source-of-truth for hostname → service mappings.
- By default the container mounts the credentials file from the node (hostPath) into `/etc/cloudflared/credentials.json`. You can instead use a sealed/encrypted Secret (SealedSecrets or SOPS) for better security.
- Update `configmap-tunnel.yaml` to change mappings, then commit to Git; Flux will apply the change.

Example mapping in `configmap-tunnel.yaml`:

```yaml
ingress:
  - hostname: portfolio.chokchai-dev.xyz
    service: http://portfolio-web.default.svc.cluster.local:3000
  - hostname: weaver-gitops.chokchai-dev.xyz
    service: http://weave-gitops.flux-system.svc.cluster.local:9001
  - hostname: rancher.chokchai-dev.xyz
    service: http://rancher.cattle-system.svc.cluster.local:80
  - service: http_status:404
```

> **Rancher UI:** deployed via Helm (`infrastructure/monitoring/rancher/`) into the
> `cattle-system` namespace with `tls=external` and the in-cluster Ingress disabled, so
> Cloudflare terminates TLS and forwards HTTP to the Rancher service. Add a `rancher`
> hostname to the tunnel in the Cloudflare dashboard, then log in with the bootstrap
> password from the HelmRelease values.

## 🔄 Image Automation

Flux automatically updates container images based on policies configured in the cluster:

**Location**: `clusters/homelab/apps/image-automation/nginx/`

1. **ImageRepository**: Monitors Docker Hub for new nginx images
2. **ImagePolicy**: Defines update rules (semantic versioning)
3. **ImageUpdateAutomation**: Automatically commits image updates to Git
4. **Deployment Annotation**: Links deployment to image policy

```yaml
# In deployment.yaml
image: nginx:1.25.2 # {"$imagepolicy": "flux-system:nginx-default"}
```

**Key Benefits:**
- **Namespace Separation**: Image automation runs in `flux-system` namespace
- **Cross-namespace Updates**: Can update applications in any namespace
- **Git-based**: All updates committed back to Git repository

## 📋 Adding New Applications

To add a new application (e.g., "api"):

### **1. Create Application Definition**
```bash
mkdir -p apps/api
```

Create the following files:
- `apps/api/deployment.yaml`
- `apps/api/service.yaml`
- `apps/api/kustomization.yaml`

### **2. Add to Apps Kustomization**
```yaml
# apps/kustomization.yaml
resources:
  - nginx
  - api        # Add new app here
```

### **3. Add Route to Ingress**
```yaml
# infrastructure/networking/ingress/ingress.yaml - Add new path to existing ingress
spec:
  rules:
    - http:
        paths:
          - path: /nginx(/|$)(.*)    # Existing nginx route
          - path: /api(/|$)(.*)      # Add new API route
            pathType: ImplementationSpecific
            backend:
              service:
                name: api-service
                port:
                  number: 8080
```

### **4. Add Image Automation (Optional)**
```bash
mkdir -p clusters/homelab/apps/image-automation/api
```

Create image automation files similar to nginx structure.

### **5. Update Image Automation Kustomization**
```yaml
# clusters/homelab/apps/image-automation/kustomization.yaml
resources:
  - nginx
  - api    # Add new app image automation
```

## 🛠️ Current Applications

### **Nginx**
- **Namespace**: `default`
- **Service Type**: `ClusterIP` (internal)
- **Ingress Path**: `/nginx` → `/` (with rewrite)
- **Cloudflare Tunnel**: Ready for external access
- **Replicas**: 1
- **Image**: `nginx:1.25.2` (auto-updated)
- **Access**: via Centralized Ingress + Cloudflare Tunnel

## 🌐 Accessing Applications

### **Via Cloudflare Tunnel (Recommended for External Access)**
The nginx service is configured to work with Cloudflare Tunnel at the `/nginx` path:

1. **Set up Cloudflare Tunnel** pointing to your MicroK8s ingress controller
2. **Access URL**: `https://yourdomain.com/nginx`
3. **Path rewriting**: `/nginx/` → `/` (handled by ingress)

Example Cloudflare Tunnel config:
```yaml
tunnel: your-tunnel-id
credentials-file: /path/to/credentials.json

ingress:
  - hostname: yourdomain.com
    service: http://MICROK8S_IP:80
  - service: http_status:404
```

### **Via Local Ingress**
Add the following to your `/etc/hosts` file:
```
<MICROK8S_NODE_IP> nginx.local
```

Then access nginx at: `http://nginx.local/nginx`

### **Via kubectl port-forward (Development)**
```bash
kubectl port-forward svc/nginx 8080:80
```
Then access at: `http://localhost:8080`

## 🔧 Maintenance

### **Manual Sync**
```bash
flux reconcile kustomization flux-system
flux reconcile kustomization infrastructure
flux reconcile kustomization image-automation
flux reconcile kustomization apps
```

### **Check Application Status**
```bash
flux get kustomizations
flux get sources git
kubectl get pods -A
```

### **View Image Automation**
```bash
flux get image repository
flux get image policy
flux get image update
```

### **Deployment Order Verification**
```bash
kubectl get kustomization -n flux-system
```
Expected order: infrastructure → image-automation → apps

## 🎉 Benefits

1. **GitOps**: All changes tracked in Git
2. **Infrastructure Separation**: Clear separation of concerns
3. **Automated**: Image updates happen automatically
4. **Scalable**: Easy to add new applications and infrastructure
5. **Proper Dependencies**: Infrastructure → Image Automation → Apps
6. **Namespace Isolation**: Infrastructure components in proper namespaces
7. **Industry Standard**: Follows Flux CD enterprise best practices

## 📚 References

- [Flux CD Documentation](https://fluxcd.io/flux/)
- [GitOps Principles](https://opengitops.dev/)
- [Kustomize Documentation](https://kustomize.io/)
- [Flux Image Automation](https://fluxcd.io/flux/components/image/)

## 🏠 Homelab Specific

This setup is optimized for homelab environments with:
- Single cluster deployment
- Proper infrastructure separation
- Cloudflare Tunnel integration
- Automatic image updates
- Enterprise-grade GitOps structure
- Simple maintenance and scaling

---

**Happy GitOps!** 🚀

## � Accessing Applications

### **Via Ingress (Recommended)**
Add the following to your `/etc/hosts` file (or DNS):
```
<MICROK8S_NODE_IP> nginx.local
```

Then access nginx at: `http://nginx.local`

### **Via kubectl port-forward (Development)**
```bash
kubectl port-forward svc/nginx 8080:80
```
Then access at: `http://localhost:8080`

## �🎉 Benefits

1. **GitOps**: All changes tracked in Git
2. **Automated**: Image updates happen automatically
3. **Scalable**: Easy to add new applications
4. **Simple**: Optimized for single environment
5. **Industry Standard**: Follows Flux CD best practices

## 📚 References

- [Flux CD Documentation](https://fluxcd.io/flux/)
- [GitOps Principles](https://opengitops.dev/)
- [Kustomize Documentation](https://kustomize.io/)

## 🏠 Homelab Specific

This setup is optimized for homelab environments with:
- Single cluster deployment
- Default namespace usage
- NodePort services for easy access
- Automatic image updates
- Simple, maintainable structure

---

**Happy GitOps!** 🚀
