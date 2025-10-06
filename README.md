# Homelab Flux Controller

A GitOps-based Kubernetes deployment setup using Flux CD for managing homelab applications with proper infrastructure separation.

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

### **6. Centralized Ingress Routing**
The ingress configuration routes traffic to all services:

**File**: `infrastructure/networking/ingress/ingress.yaml`
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
