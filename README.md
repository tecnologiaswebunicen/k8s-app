# Kubernetes Hello World Application with Argo CD

This repository demonstrates a GitOps workflow using Argo CD to deploy a simple "Hello World" application to a Kubernetes cluster running on Docker Desktop.

## 📋 Overview

This project showcases:
- **GitOps principles** using Argo CD as a continuous deployment tool
- Deployment of Google's `hello-app` sample application
- Kubernetes manifest management (Deployment and Service)
- Local development workflow with Docker Desktop Kubernetes

## 🏗️ Repository Structure

```
.
├── README.md           # This file
├── application.md      # Detailed application deployment guide
├── argocd.md          # Argo CD installation and setup guide
├── HPA-TESTING.md     # Horizontal Pod Autoscaler testing guide
├── deployment.yaml    # Kubernetes Deployment manifest (2 replicas)
├── service.yaml       # Kubernetes Service manifest (NodePort)
├── hpa.yaml           # Horizontal Pod Autoscaler configuration
└── load-generator.yaml # Load generator for HPA testing
```

## 🎯 What Gets Deployed

- **Application**: Google's `hello-app:1.0` - A simple Go application displaying "Hello, world!" with version and hostname information
- **Replicas**: 2 pods for high availability
- **Service Type**: NodePort for external access
- **Resources**: 
  - Requests: 64Mi memory, 100m CPU
  - Limits: 128Mi memory, 200m CPU

## 🚀 Quick Start

### Prerequisites

- Docker Desktop with Kubernetes enabled
- `kubectl` configured for Docker Desktop cluster
- (Optional) Argo CD CLI installed

### 1. Install Argo CD

Follow the instructions in [`argocd.md`](./argocd.md):

```bash
# Create namespace
kubectl create namespace argocd

# Install Argo CD
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Access the UI
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

### 2. Deploy the Application

**Option A: Using Argo CD CLI**
```bash
# Login to Argo CD
argocd login localhost:8080

# Create application
argocd app create hello-world \
  --repo https://github.com/tecnologiaswebunicen/k8s-app.git \
  --path . \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace default

# Sync application
argocd app sync hello-world
```

**Option B: Using Argo CD UI**
1. Open https://localhost:8080
2. Click **+ NEW APP**
3. Configure with repository details
4. Click **CREATE** and **SYNC**

**Option C: Direct kubectl (bypasses GitOps)**
```bash
kubectl apply -f deployment.yaml
kubectl apply -f service.yaml
```

### 3. Access the Application

```bash
# Port forward to the service
kubectl port-forward svc/hello-world 8081:80

# Access in browser
open http://localhost:8081

# Or use NodePort
kubectl get svc hello-world -o jsonpath='{.spec.ports[0].nodePort}'
# Then access http://localhost:<NODE_PORT>
```

## 📚 Documentation

- **[argocd.md](./argocd.md)**: Complete Argo CD installation guide for Docker Desktop
- **[application.md](./application.md)**: Comprehensive application deployment guide with both CLI and UI methods
- **[HPA-TESTING.md](./HPA-TESTING.md)**: Guide to testing Horizontal Pod Autoscaler with load generation

## 🔄 GitOps Workflow

1. **Modify**: Edit the Kubernetes manifests (e.g., change replica count in `deployment.yaml`)
2. **Commit**: Push changes to Git repository
3. **Sync**: Argo CD detects changes and shows "OutOfSync" status
4. **Deploy**: Manually sync (or auto-sync if enabled) to apply changes

### Example: Scale the Application

```bash
# Edit deployment.yaml - change replicas from 2 to 3
git add deployment.yaml
git commit -m "Scale to 3 replicas"
git push

# Sync with Argo CD
argocd app sync hello-world

# Verify
kubectl get pods -l app=hello-world
```

## 🧪 Testing

```bash
# Check pod status
kubectl get pods -l app=hello-world

# View logs
kubectl logs -l app=hello-world --tail=50

# Test application
curl http://localhost:8081

# Test load balancing (via NodePort)
for i in {1..10}; do curl -s http://localhost:<NODE_PORT> | grep Hostname; done
```

## 🛠️ Troubleshooting

| Issue | Solution |
|-------|----------|
| Application OutOfSync | Run `argocd app sync hello-world` |
| Pods not running | Check with `kubectl describe pod <pod-name>` |
| Cannot access UI | Restart port forwarding: `kubectl port-forward svc/argocd-server -n argocd 8080:443` |
| Cannot access app | Verify service and port forwarding: `kubectl get svc hello-world` |

## 🧹 Cleanup

```bash
# Delete application via Argo CD
argocd app delete hello-world

# Or directly via kubectl
kubectl delete -f deployment.yaml -f service.yaml

# Optional: Uninstall Argo CD
kubectl delete -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl delete namespace argocd
```

## 📖 Learning Objectives

This repository helps you learn:
- ✅ Setting up Argo CD on a local Kubernetes cluster
- ✅ Creating and managing Argo CD applications
- ✅ Implementing GitOps workflows
- ✅ Deploying containerized applications with Kubernetes
- ✅ Using Kubernetes Services for application access
- ✅ Scaling applications declaratively
- ✅ Testing Horizontal Pod Autoscaler (HPA) with metrics-based autoscaling
- ✅ Load testing and monitoring Kubernetes applications

## 🔗 Resources

- [Argo CD Documentation](https://argo-cd.readthedocs.io/)
- [Docker Desktop Kubernetes](https://docs.docker.com/desktop/kubernetes/)
- [Kubernetes Documentation](https://kubernetes.io/docs/)
- [Google Hello App Source](https://github.com/GoogleCloudPlatform/kubernetes-engine-samples/tree/main/quickstarts/hello-app)

## 📝 License

This is an educational repository for learning Kubernetes and GitOps principles.

## 👥 Contributing

This repository is maintained by [tecnologiaswebunicen](https://github.com/tecnologiaswebunicen).
