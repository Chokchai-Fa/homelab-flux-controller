# Homelab Flux Controller

A GitOps-based Kubernetes deployment setup using Flux CD for managing homelab applications.

## 🏗️ Project Structure

This repository follows GitOps best practices with a clear separation between cluster configurations and application definitions.

```
homelab-flux-controller/
├── README.md                           # This file
├── apps/                              # Application definitions (WHAT to deploy)
│   ├── kustomization.yaml             # Main apps orchestration
│   └── nginx/                         # Example nginx application
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

To add a new application (e.g., "webapp"):

### **1. Create Application Definition**
```bash
mkdir -p apps/webapp
```

Create the following files:
- `apps/webapp/deployment.yaml`
- `apps/webapp/service.yaml`
- `apps/webapp/kustomization.yaml`
- `apps/webapp/image-automation/` (if needed)

### **2. Add to Apps Kustomization**
```yaml
# apps/kustomization.yaml
resources:
  - nginx
  - webapp  # Add new app here
```

### **3. Create Cluster Deployment Configuration**
```yaml
# clusters/homelab/apps/webapp.yaml
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: webapp
  namespace: flux-system
spec:
  interval: 5m
  path: ./apps/webapp
  prune: true
  sourceRef:
    kind: GitRepository
    name: flux-system
```

### **4. Add to Cluster Apps Kustomization**
```yaml
# clusters/homelab/apps/kustomization.yaml
resources:
  - nginx.yaml
  - webapp.yaml  # Add new app deployment
```

## 🛠️ Current Applications

### **Nginx**
- **Namespace**: `default`
- **Service Type**: `NodePort` (port 30080)
- **Replicas**: 1
- **Image**: `nginx:1.25.2` (auto-updated)

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

## 🎉 Benefits

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
