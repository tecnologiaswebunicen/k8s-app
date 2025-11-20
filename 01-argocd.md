# Using Argo CD with Docker Desktop Kubernetes

Using Argo CD with the Kubernetes cluster provided by Docker Desktop allows you to experiment with GitOps on your local machine. This setup provides a seamless, self-contained environment for managing your applications directly from a Git repository.

Here is a step-by-step guide to installing Argo CD on Docker Desktop.

---

## Prerequisites

- **Docker Desktop**: Ensure you have Docker Desktop installed on your system.
- **Kubernetes Enabled**: In Docker Desktop, go to **Settings > Kubernetes** and check **Enable Kubernetes**.
- **kubectl**: The Kubernetes command-line tool `kubectl` must be installed and configured to communicate with your Docker Desktop cluster.

---

## Step 1: Install Argo CD on Your Cluster

Install Argo CD by applying its installation manifest to your local Kubernetes cluster, which will create necessary resources in an `argocd` namespace.

### Create the `argocd` namespace:
```sh
kubectl create namespace argocd
```

### Apply the installation manifest:
```sh
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

### Verify that Argo CD pods are running:
```sh
kubectl get pods -n argocd
```

---

## Step 2: Access the Argo CD UI

Use port forwarding to access the Argo CD UI from your browser.

### Set up port forwarding:
```sh
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

Open [https://localhost:8080](https://localhost:8080) in your browser. You may see a security warning due to a self-signed certificate, which can be ignored for local development.

---

## Step 3: Log in to the Argo CD UI

Retrieve the initial admin password from a Kubernetes secret to log in.

### Get the password:
```sh
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 --decode; echo
```

Log in using:

- **Username**: `admin`
- **Password**: (value from the command above)

---

## Step 4: Create a GitOps Application

Create a new application in the Argo CD UI that pulls manifests from a Git repository.

1. Click **+ NEW APP**.
2. Configure the application using, for example, the *guestbook* application from the ArgoProj example apps repository.
3. Specify:
   - **Repository URL**
   - **Revision (e.g., HEAD or main)**
   - **Path within the repo**
   - **Destination cluster** (e.g., `https://kubernetes.default.svc`)
   - **Namespace**

4. Click **Create**.

---

## Step 5: Sync and View Your Application

After creating the application, sync it to deploy to your cluster.

1. Select your application in the UI.
2. Click **Sync**.
3. Once the sync is complete, the application status should show:

   - `Synced`
   - `Healthy`

Indicating that the resources are successfully deployed and running.

---