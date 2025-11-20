# Testing Horizontal Pod Autoscaler (HPA) with Hello World App

This guide demonstrates how to test Kubernetes Horizontal Pod Autoscaler (HPA) with the hello-world application on Docker Desktop.

---

## 📋 Prerequisites

- Docker Desktop with Kubernetes enabled
- Application deployed (see [application.md](./application.md))
- Metrics Server installed (see Step 1 below)
- `kubectl` configured for your cluster

---

## 🎯 What is HPA?

Horizontal Pod Autoscaler (HPA) automatically scales the number of pods in a deployment based on observed metrics like:
- **CPU utilization**
- **Memory utilization**
- **Custom metrics** (e.g., requests per second)

HPA helps maintain optimal application performance during traffic spikes while reducing costs during low-traffic periods.

---

## 🚀 Step 1: Install Metrics Server

Metrics Server is required for HPA to collect resource metrics from pods.

### Install on Docker Desktop

```bash
# Download the metrics-server manifest
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

# Patch metrics-server for Docker Desktop (required for local development)
kubectl patch deployment metrics-server -n kube-system --type='json' \
  -p='[{"op": "add", "path": "/spec/template/spec/containers/0/args/-", "value": "--kubelet-insecure-tls"}]'
```

### Verify Installation

```bash
# Check metrics-server is running
kubectl get deployment metrics-server -n kube-system

# Wait for metrics to be available (may take 1-2 minutes)
kubectl top nodes

# Check pod metrics
kubectl top pods -n default
```

If you see CPU and memory metrics, Metrics Server is working correctly!

---

## 🔧 Step 2: Understand the HPA Configuration

The `hpa.yaml` file defines autoscaling rules:

```yaml
minReplicas: 2        # Minimum pods (never scale below this)
maxReplicas: 10       # Maximum pods (never scale above this)

metrics:
  - CPU: 50%          # Scale up when CPU > 50% of requested CPU
  - Memory: 70%       # Scale up when Memory > 70% of requested memory

behavior:
  scaleUp: Fast       # Scale up quickly during spikes
  scaleDown: Gradual  # Scale down slowly to avoid flapping
```

**Key Point**: HPA uses the `resources.requests` values from `deployment.yaml` as the baseline:
- CPU request: 100m (0.1 CPU cores)
- Memory request: 64Mi

So 50% CPU utilization = 50m of CPU usage per pod.

---

## 📦 Step 3: Deploy HPA

### Apply the HPA Resource

```bash
# Apply the HPA configuration
kubectl apply -f hpa.yaml

# Verify HPA is created
kubectl get hpa

# View detailed HPA status
kubectl describe hpa hello-world-hpa
```

You should see output like:
```
NAME               REFERENCE                TARGETS                        MINPODS   MAXPODS   REPLICAS   AGE
hello-world-hpa    Deployment/hello-world   <unknown>/50%, <unknown>/70%   2         10        2          10s
```

The `<unknown>` will change to actual percentages once metrics are collected (wait ~1 minute).

### Monitor HPA Status

```bash
# Watch HPA in real-time
kubectl get hpa hello-world-hpa --watch

# Or use this for more details
watch -n 2 kubectl get hpa hello-world-hpa
```

---

## 🔥 Step 4: Generate Load to Trigger Autoscaling

Now we'll generate CPU load to trigger the HPA to scale up.

### Method A: Using Load Generator Pod (Recommended)

This creates a pod that continuously sends requests to your application:

```bash
# Deploy the load generator
kubectl apply -f load-generator.yaml

# Verify it's running
kubectl get pod load-generator

# View load generator logs
kubectl logs -f load-generator
```

### Method B: Using Multiple Terminal Windows

If you prefer manual control, open multiple terminals and run:

```bash
# Terminal 1, 2, 3, etc. - Run this in each
kubectl run -i --tty load-generator-$RANDOM --rm --image=busybox --restart=Never -- /bin/sh -c "while sleep 0.01; do wget -q -O- http://hello-world.default.svc.cluster.local; done"
```

### Method C: Using Apache Bench (if installed)

```bash
# Install apache bench
brew install apache2  # macOS

# Generate load (10 concurrent connections, 100000 requests)
ab -n 100000 -c 10 http://localhost:<NODE_PORT>/
```

---

## 📊 Step 5: Monitor the Autoscaling

### Watch Pods Scale Up

Open multiple terminal windows to monitor different aspects:

**Terminal 1: Watch HPA**
```bash
kubectl get hpa hello-world-hpa --watch
```

**Terminal 2: Watch Pods**
```bash
kubectl get pods -l app=hello-world --watch
```

**Terminal 3: Watch CPU/Memory Usage**
```bash
watch -n 2 kubectl top pods -l app=hello-world
```

**Terminal 4: View HPA Events**
```bash
kubectl get events --sort-by='.lastTimestamp' | grep HorizontalPodAutoscaler
```

### What to Expect

1. **Initial State**: 2 pods running (minReplicas)
2. **After 30-60 seconds**: CPU usage increases above 50%
3. **HPA Triggers**: New pods start being created
4. **Scaling Up**: Pods increase to 3, 4, 5... up to 10 (maxReplicas)
5. **Load Distribution**: CPU usage per pod decreases as load is distributed

Example output:
```
NAME               REFERENCE                TARGETS          MINPODS   MAXPODS   REPLICAS   AGE
hello-world-hpa    Deployment/hello-world   125%/50%, 23%/70%   2         10        2          2m
hello-world-hpa    Deployment/hello-world   125%/50%, 23%/70%   2         10        4          2m30s
hello-world-hpa    Deployment/hello-world   85%/50%, 20%/70%    2         10        6          3m
hello-world-hpa    Deployment/hello-world   45%/50%, 18%/70%    2         10        6          3m30s
```

---

## 🔽 Step 6: Test Scale Down

### Stop the Load

```bash
# Delete the load generator pod
kubectl delete pod load-generator

# Or if using Method B, press Ctrl+C in all terminals
```

### Watch Pods Scale Down

```bash
# Monitor the scale down (will take ~5 minutes due to stabilization window)
kubectl get hpa hello-world-hpa --watch
```

HPA will:
1. Detect CPU usage has dropped below 50%
2. Wait for the stabilization window (60 seconds)
3. Gradually scale down pods (max 50% at a time)
4. Eventually return to minReplicas (2 pods)

---

## 📈 Step 7: Advanced Testing

### Test Memory-Based Autoscaling

The current hello-app doesn't consume much memory. To test memory-based scaling, you'd need:

1. An app that allocates memory based on requests
2. Or modify the HPA to use lower memory thresholds

### View HPA Metrics in Detail

```bash
# Get current metrics for the deployment
kubectl get --raw "/apis/metrics.k8s.io/v1beta1/namespaces/default/pods" | jq '.'

# View HPA calculations
kubectl describe hpa hello-world-hpa
```

### Simulate Different Load Patterns

**Gradual Load Increase:**
```bash
# Start with 1 load generator, then add more
kubectl apply -f load-generator.yaml
sleep 60
kubectl run load-generator-2 --image=busybox --restart=Never -- /bin/sh -c "while sleep 0.01; do wget -q -O- http://hello-world.default.svc.cluster.local; done"
```

**Spike Load:**
```bash
# Run 5 load generators simultaneously
for i in {1..5}; do
  kubectl run load-generator-$i --image=busybox --restart=Never -- /bin/sh -c "while sleep 0.01; do wget -q -O- http://hello-world.default.svc.cluster.local; done" &
done
```

---

## 🎯 Step 8: Testing with Argo CD

### Apply HPA via Argo CD

If you're using Argo CD (GitOps approach):

```bash
# Commit the HPA configuration
git add hpa.yaml
git commit -m "Add HPA configuration"
git push

# Sync the application
argocd app sync hello-world

# Or wait for auto-sync if enabled
```

### Monitor HPA in Argo CD UI

1. Open Argo CD UI: https://localhost:8080
2. Click on `hello-world` application
3. You'll see the HPA resource in the application tree
4. Click on HPA to view its status and events

**Note**: The Deployment's replica count may show as "OutOfSync" when HPA is scaling, which is normal behavior. The HPA manages replicas dynamically, overriding the static value in `deployment.yaml`.

---

## 🧪 Expected Results

### Successful HPA Test Checklist

- ✅ Metrics Server installed and reporting metrics
- ✅ HPA shows current CPU/memory usage (not `<unknown>`)
- ✅ Under load: CPU usage > 50%, pods scale from 2 → up to 10
- ✅ Load distributed: Individual pod CPU usage decreases
- ✅ After stopping load: Pods scale down gradually to 2
- ✅ No pod crashes or restarts during scaling

### Sample Timeline

```
0:00  - Start: 2 pods @ 5% CPU
0:30  - Apply load generator
1:00  - 2 pods @ 120% CPU → HPA decides to scale
1:30  - 4 pods @ 80% CPU → HPA adds more
2:00  - 6 pods @ 55% CPU → HPA adds more
2:30  - 8 pods @ 45% CPU → Stable
5:00  - Stop load
6:00  - 8 pods @ 5% CPU → Stabilization window
7:00  - 4 pods @ 10% CPU → Scaled down 50%
8:00  - 2 pods @ 15% CPU → Back to minimum
```

---

## 🛠️ Troubleshooting

### HPA Shows `<unknown>` for Metrics

**Problem**: HPA can't read metrics from Metrics Server.

**Solutions**:
```bash
# Check Metrics Server is running
kubectl get pods -n kube-system | grep metrics-server

# Verify metrics are available
kubectl top pods

# Restart Metrics Server if needed
kubectl rollout restart deployment metrics-server -n kube-system

# Wait 1-2 minutes for metrics to be collected
```

### HPA Not Scaling Up

**Problem**: Pods aren't increasing despite high CPU.

**Checks**:
```bash
# Verify HPA is watching the correct deployment
kubectl describe hpa hello-world-hpa

# Check if deployment has resource requests defined
kubectl get deployment hello-world -o yaml | grep -A 5 resources

# View HPA events for error messages
kubectl describe hpa hello-world-hpa | grep Events -A 10
```

**Common Cause**: If `resources.requests` are not defined in `deployment.yaml`, HPA cannot calculate percentages.

### Pods Scale Up But Not Down

**Problem**: Pods remain at high count after load stops.

**Explanation**: This is by design! The `stabilizationWindowSeconds: 60` prevents rapid scale-down to avoid "flapping" (constantly scaling up and down).

**Solution**: Wait 5-10 minutes, or reduce the stabilization window in `hpa.yaml`.

### CPU Usage Stays Low Even With Load

**Problem**: Load generator is running but CPU stays at 5%.

**Solutions**:
```bash
# Increase load by adding more generators
kubectl scale deployment load-generator --replicas=5

# Or use the spike load script from Step 7
```

### Metrics Server Crashes on Docker Desktop

**Problem**: Metrics Server pod keeps restarting.

**Solution**: Ensure you applied the insecure TLS patch:
```bash
kubectl patch deployment metrics-server -n kube-system --type='json' \
  -p='[{"op": "add", "path": "/spec/template/spec/containers/0/args/-", "value": "--kubelet-insecure-tls"}]'
```

---

## 📊 Understanding HPA Calculations

### How HPA Calculates Desired Replicas

```
desiredReplicas = ceil[currentReplicas * (currentMetricValue / targetMetricValue)]
```

**Example**:
- Current replicas: 2
- Current CPU usage: 125% (of requested)
- Target CPU usage: 50%
- Desired replicas = ceil[2 * (125 / 50)] = ceil[5] = 5 pods

### Why Use Percentage vs Absolute Values?

The HPA uses **percentage of requested resources** because:
- ✅ Scales proportionally to pod resource needs
- ✅ Works across different pod sizes
- ✅ Easier to reason about ("scale when pods are half-full")

---

## 🧹 Cleanup

When you're done testing:

```bash
# Delete the load generator
kubectl delete pod load-generator

# Delete any other load generators
kubectl delete pods -l app=load-generator

# Delete the HPA (pods will remain at current count)
kubectl delete -f hpa.yaml

# Or delete HPA and reset to 2 replicas
kubectl delete hpa hello-world-hpa
kubectl scale deployment hello-world --replicas=2

# Optional: Uninstall Metrics Server
kubectl delete -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```

---

## 🎓 Key Takeaways

1. **Metrics Server is required** for HPA to function
2. **Resource requests must be defined** in deployment for HPA percentage calculations
3. **HPA scaling is gradual** to prevent flapping
4. **Scale-up is fast, scale-down is slow** by design
5. **HPA overrides deployment replica count** dynamically
6. **Multiple metrics** (CPU + Memory) provide better scaling decisions
7. **Stabilization windows** prevent unnecessary scaling events

---

## 🔗 Next Steps

- **Custom Metrics**: Use Prometheus Adapter for custom metrics (requests/sec, queue length)
- **Vertical Pod Autoscaler (VPA)**: Automatically adjust resource requests/limits
- **Cluster Autoscaler**: Scale the cluster nodes themselves (requires cloud provider)
- **HPA v2 Features**: Explore behavior policies, multiple metrics, and external metrics

---

## 📚 Resources

- [Kubernetes HPA Documentation](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/)
- [Metrics Server GitHub](https://github.com/kubernetes-sigs/metrics-server)
- [HPA Walkthrough](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale-walkthrough/)
- [HPA Best Practices](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/#best-practices)
