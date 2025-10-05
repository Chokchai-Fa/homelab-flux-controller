# Homelab Flux Controller

A GitOps-based Kubernetes deployment setup using Flux CD for managing homelab applications.

## 🏗️ Project Structure

This repository follows GitOps b## 🌐 Accessing Applications

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
Then access at: `http://localhost:8080`ith a clear separation between cluster configurations and application definitions.

```
homelab-flux-controller/
├── README.md                           # This file
├── apps/                              # Application definitions (WHAT to deploy)
│   ├── kustomization.yaml             # Main apps orchestration
│   ├── ingress/                       # Centralized ingress routing
│   │   ├── ingress.yaml               # Multi-service ingress configuration
│   │   └── kustomization.yaml         # Ingress resource management
│   └── nginx/                         # Nginx web server application
│       ├── deployment.yaml            # Kubernetes Deployment manifest
│       ├── service.yaml               # Kubernetes Service manifest
│       ├── kustomization.yaml         # App resource organization
│       └── image-automation/          # Flux image automation
│           ├── kustomization.yaml
│           ├── nginx-repository.yaml   # Image repository to watch
│           ├── nginx-image-policy.yaml # Image update policy
│           └── nginx-image-update.yaml # Image update automation
└── clusters/                          # Cluster configurations (HOW to deploy)
    └── homelab/                       # Homelab cluster
        ├── kustomization.yaml         # Cluster orchestration
        ├── flux-system/               # Flux CD core components
        │   ├── gotk-components.yaml   # Flux controllers
        │   ├── gotk-sync.yaml         # Git sync configuration
        │   └── kustomization.yaml
        └── apps/                      # Cluster app deployments
            ├── kustomization.yaml
            └── nginx.yaml             # How to deploy nginx to this cluster
```

## 🎯 Design Principles

### **Separation of Concerns**
- **`apps/`**: Contains application definitions (Kubernetes manifests, configurations)
- **`apps/ingress/`**: Centralized ingress routing for all microservices
- **`clusters/`**: Contains deployment instructions (Flux Kustomizations, cluster-specific settings)

### **GitOps Flow**
```
Git Repository → Flux System → Cluster Apps → Application Manifests → Kubernetes Resources
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
The cluster kustomization orchestrates all cluster resources:

**File**: `clusters/homelab/kustomization.yaml`
```yaml
resources:
  - flux-system    # Flux core components
  - apps          # Application deployments
```

### **3. Application Deployment**
Each app has a deployment configuration in the cluster:

**File**: `clusters/homelab/apps/nginx.yaml`
```yaml
spec:
  path: ./apps/nginx    # Points to app definition
```

### **4. Application Definition**
The app directory contains all application resources:

**File**: `apps/nginx/kustomization.yaml`
```yaml
resources:
  - deployment.yaml      # Kubernetes manifests
  - service.yaml
  - image-automation     # Automated image updates
```

### **5. Centralized Ingress Routing**
The ingress configuration routes traffic to all services:

**File**: `apps/ingress/ingress.yaml`
```yaml
spec:
  rules:
    - http:
        paths:
          - path: /nginx(/|$)(.*)    # Routes /nginx to nginx service
            backend:
              service:
                name: nginx
```

## 🔄 Image Automation

Flux automatically updates container images based on policies:

1. **ImageRepository**: Monitors Docker Hub for new nginx images
2. **ImagePolicy**: Defines update rules (semantic versioning)
3. **ImageUpdateAutomation**: Automatically commits image updates to Git
4. **Deployment Annotation**: Links deployment to image policy

```yaml
# In deployment.yaml
image: nginx:1.25.2 # {"$imagepolicy": "flux-system:nginx-default"}
```

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
- `apps/api/image-automation/` (if needed)

### **2. Add to Apps Kustomization**
```yaml
# apps/kustomization.yaml
resources:
  - ingress    # Centralized routing
  - nginx
  - api        # Add new app here
```

### **3. Add Route to Ingress**
```yaml
# apps/ingress/ingress.yaml - Add new path to existing ingress
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

### **4. Create Cluster Deployment Configuration**
```yaml
# clusters/homelab/apps/api.yaml
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: api
  namespace: flux-system
spec:
  interval: 5m
  path: ./apps/api
  prune: true
  sourceRef:
    kind: GitRepository
    name: flux-system
```

### **5. Add to Cluster Apps Kustomization**
```yaml
# clusters/homelab/apps/kustomization.yaml
resources:
  - nginx.yaml
  - api.yaml  # Add new app deployment
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

## 🔧 Maintenance

### **Manual Sync**
```bash
flux reconcile kustomization flux-system
```

### **Check Application Status**
```bash
flux get kustomizations
flux get helmreleases
```

### **View Image Automation**
```bash
flux get image repository
flux get image policy
```

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
