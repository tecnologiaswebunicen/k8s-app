# Deploying a Sample Application with Argo CD

This guide demonstrates how to deploy the `tutum/hello-world` application to your Docker Desktop Kubernetes cluster using Argo CD. This builds upon the Argo CD installation from **Step 01**.

The `tutum/hello-world` is a simple Docker image that runs an nginx web server displaying hostname and connection information - perfect for demonstrating GitOps workflows with Argo CD.

---

## Prerequisites

- Completed [Step 01: Argo CD Installation](../01-argocd/argo-cd-docker-desktop.md)
- Argo CD is running on your Docker Desktop Kubernetes cluster
- `kubectl` is configured to access your cluster
- Port forwarding to Argo CD UI is active (or re-run the command below)

---

## Overview

We'll deploy a sample application using two methods:

1. **Method A**: Using the Argo CD CLI (declarative approach)
2. **Method B**: Using the Argo CD UI (manual approach)

Both methods use the same Kubernetes manifests located in this directory:
- `deployment.yaml` - Defines the hello-world deployment with 2 replicas
- `service.yaml` - Exposes the application via NodePort

---

## Method A: Deploy Using Argo CD CLI

### Step 1: Install Argo CD CLI (Optional)

If you haven't already installed the Argo CD CLI:

**macOS (using Homebrew):**
```sh
brew install argocd
```

**macOS (manual download):**
```sh
curl -sSL -o /usr/local/bin/argocd https://github.com/argoproj/argo-cd/releases/latest/download/argocd-darwin-amd64
chmod +x /usr/local/bin/argocd
```

### Step 2: Login to Argo CD

First, ensure port forwarding is running:
```sh
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

In a new terminal, login using the CLI:
```sh
argocd login localhost:8080
```

When prompted:
- **Username**: `admin`
- **Password**: (retrieve using the command from Step 01)
  ```sh
  kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 --decode; echo
  ```

You may need to accept the certificate warning by typing `y`.

### Step 3: Create the Application

Create an Argo CD application pointing to this directory in your Git repository:

```sh
argocd app create hello-world \
  --repo https://github.com/YOUR_USERNAME/YOUR_REPO.git \
  --path 02-application \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace default
```

**Important**: Replace `YOUR_USERNAME` and `YOUR_REPO` with your actual Git repository details.

If you haven't pushed this to Git yet, see the **Local Deployment** section below.

### Step 4: Sync the Application

Deploy the application to your cluster:
```sh
argocd app sync hello-world
```

### Step 5: Check Application Status

View the application status:
```sh
argocd app get hello-world
```

You should see:
- **Sync Status**: `Synced`
- **Health Status**: `Healthy`

---

## Method B: Deploy Using Argo CD UI

### Step 1: Access the Argo CD UI

Ensure port forwarding is running:
```sh
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

Open [https://localhost:8080](https://localhost:8080) and login with:
- **Username**: `admin`
- **Password**: (from Step 01)

### Step 2: Create a New Application

1. Click **+ NEW APP** button
2. Fill in the **GENERAL** section:
   - **Application Name**: `hello-world`
   - **Project**: `default`
   - **Sync Policy**: `Manual` (or `Automatic` for auto-sync)

3. Fill in the **SOURCE** section:
   - **Repository URL**: Your Git repository URL (e.g., `https://github.com/YOUR_USERNAME/YOUR_REPO.git`)
   - **Revision**: `HEAD` or `main`
   - **Path**: `02-application`

4. Fill in the **DESTINATION** section:
   - **Cluster URL**: `https://kubernetes.default.svc`
   - **Namespace**: `default`

5. Click **CREATE** at the top

### Step 3: Sync the Application

1. Click on your `hello-world` application in the list
2. Click the **SYNC** button
3. Review the resources and click **SYNCHRONIZE**

### Step 4: View Application Details

Once synced, you'll see a visual representation of your application resources:
- **Deployment**: hello-world
- **ReplicaSet**: Created by the deployment
- **Pods**: 2 running pods
- **Service**: hello-world service

All resources should show as **Healthy** and **Synced**.

---

## Local Deployment (Without Git Repository)

If you want to test locally without pushing to Git, you can deploy directly using kubectl:

```sh
kubectl apply -f 02-application/deployment.yaml
kubectl apply -f 02-application/service.yaml
```

However, this bypasses Argo CD and doesn't demonstrate GitOps principles.

---

## Verify the Deployment

### Check Pod Status

```sh
kubectl get pods -l app=hello-world
```

You should see 2 pods running:
```
NAME                           READY   STATUS    RESTARTS   AGE
hello-world-xxxxxxxxxx-xxxxx   1/1     Running   0          1m
hello-world-xxxxxxxxxx-xxxxx   1/1     Running   0          1m
```

### Check Service

```sh
kubectl get svc hello-world
```

Note the **NodePort** assigned to the service (e.g., `30000-32767` range).

### Access the Application

Since we're using Docker Desktop, you can access the application via:

```sh
kubectl port-forward svc/hello-world 8081:80
```

Then open [http://localhost:8081](http://localhost:8081) in your browser.

You should see a page displaying:
- Hostname (pod name)
- Container information
- Request details

**Alternatively**, get the NodePort and access directly:
```sh
kubectl get svc hello-world -o jsonpath='{.spec.ports[0].nodePort}'
```

Then visit `http://localhost:<NODE_PORT>` (Docker Desktop forwards NodePorts automatically).

---

## Test GitOps Workflow

To see Argo CD's GitOps capabilities in action:

### Step 1: Modify the Application

Edit `deployment.yaml` and change the replica count:
```yaml
spec:
  replicas: 3  # Changed from 2 to 3
```

### Step 2: Commit and Push

```sh
git add 02-application/deployment.yaml
git commit -m "Scale hello-world to 3 replicas"
git push
```

### Step 3: Sync in Argo CD

**Via UI:**
- Click on the `hello-world` application
- Notice it shows **OutOfSync**
- Click **SYNC** to apply changes

**Via CLI:**
```sh
argocd app sync hello-world
```

### Step 4: Verify the Change

```sh
kubectl get pods -l app=hello-world
```

You should now see 3 pods running!

---

## Managing the Application

### View Application Logs

**Via UI:**
- Click on a pod in the application view
- Click **LOGS** tab

**Via kubectl:**
```sh
kubectl logs -l app=hello-world --tail=50
```

### View Application Details

```sh
argocd app get hello-world --refresh
```

### Delete the Application

**Via CLI:**
```sh
argocd app delete hello-world
```

**Via UI:**
- Click on the application
- Click **DELETE** button
- Confirm deletion

This will remove the application from Argo CD and delete all associated Kubernetes resources.

---

## Troubleshooting

### Application Shows as "OutOfSync"

This is normal if you haven't synced yet, or if there are changes in Git that aren't applied to the cluster.

**Solution**: Click **SYNC** in the UI or run `argocd app sync hello-world`

### Application Shows as "Degraded"

Check pod status:
```sh
kubectl get pods -l app=hello-world
kubectl describe pod <pod-name>
```

Common issues:
- Image pull errors (check internet connection)
- Resource constraints (unlikely with this small app)

### Cannot Access Application in Browser

1. Verify pods are running: `kubectl get pods -l app=hello-world`
2. Verify service exists: `kubectl get svc hello-world`
3. Check port forwarding is active: `kubectl port-forward svc/hello-world 8081:80`
4. Try accessing: `http://localhost:8081`

### Argo CD UI Not Accessible

Restart port forwarding:
```sh
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

---

## Next Steps

Now that you've successfully deployed an application with Argo CD, you can:

1. **Explore Auto-Sync**: Enable automatic synchronization for continuous deployment
2. **Add Health Checks**: Configure custom health checks for your applications
3. **Try Rollbacks**: Use Argo CD to rollback to previous versions
4. **Multi-Environment**: Set up different applications for dev/staging/prod
5. **Helm Charts**: Deploy applications using Helm charts instead of plain manifests
6. **Kustomize**: Use Kustomize for environment-specific configurations

---

## Clean Up

When you're done experimenting:

### Delete the Application
```sh
argocd app delete hello-world
# or via kubectl
kubectl delete -f 02-application/
```

### Stop Port Forwarding
Press `Ctrl+C` in the terminals running port-forward commands.

### Optional: Uninstall Argo CD
```sh
kubectl delete -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl delete namespace argocd
```

---

## Summary

You've learned how to:
- ✅ Deploy an application using Argo CD (both CLI and UI methods)
- ✅ Verify application health and sync status
- ✅ Access your deployed application
- ✅ Test the GitOps workflow by making changes
- ✅ Manage and troubleshoot Argo CD applications

This workflow demonstrates the core GitOps principle: **Git as the single source of truth** for your infrastructure and applications.
