# Istio Ambient Mode Quick Reference

## 🎯 What You Have Now

After running the playbook:
- ✅ Kubernetes cluster with Cilium CNI
- ✅ Istio control plane (istiod)
- ✅ ztunnel DaemonSet (L4 proxy on each node)
- ✅ Test namespace with ambient mode enabled

## 🚀 Enable Ambient Mode for Your Apps

### Step 1: Label Your Namespace
```bash
kubectl label namespace my-app istio.io/dataplane-mode=ambient
```

That's it! Any pod in this namespace is now in the mesh.

### Step 2: Deploy Your Application
```bash
kubectl apply -f my-app.yaml -n my-app
```

No special annotations needed. No sidecar injection. Just deploy normally.

### Step 3: Verify Mesh Enrollment
```bash
# Check namespace label
kubectl get namespace my-app -o jsonpath='{.metadata.labels}'

# Verify NO sidecar (should show 1/1, not 2/2)
kubectl get pods -n my-app

# Check ztunnel is capturing traffic
kubectl logs -n istio-system -l app=ztunnel | grep my-app
```

## 🔒 What You Get Automatically

With just the namespace label:
- ✅ **mTLS encryption** between all pods (zero-trust networking)
- ✅ **L4 authorization policies** (allow/deny based on workload identity)
- ✅ **Telemetry** (metrics for connections, bytes transferred)
- ✅ **No performance overhead** in pods (ztunnel is shared)

## 📊 Observability

View traffic metrics:
```bash
# Install Kiali for visualization
kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-1.21/samples/addons/kiali.yaml

# Port-forward to access
kubectl port-forward -n istio-system svc/kiali 20001:20001

# Open http://localhost:20001 in browser
```

Check metrics:
```bash
# Install Prometheus
kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-1.21/samples/addons/prometheus.yaml

# Query Istio metrics
kubectl port-forward -n istio-system svc/prometheus 9090:9090
```

## 🎛️ L7 Features (Optional)

For advanced routing (headers, paths, retries), deploy a waypoint proxy:

```bash
# Deploy waypoint for specific namespace
istioctl waypoint apply -n my-app

# Or for specific service
istioctl waypoint apply -n my-app --for service/my-service
```

Now you can use L7 features:
```yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: my-service-routes
  namespace: my-app
spec:
  hosts:
  - my-service
  http:
  - match:
    - headers:
        version:
          exact: v2
    route:
    - destination:
        host: my-service
        subset: v2
  - route:
    - destination:
        host: my-service
        subset: v1
```

## 🔐 Security Policies

### Allow only specific services to communicate:
```yaml
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: frontend-policy
  namespace: my-app
spec:
  selector:
    matchLabels:
      app: frontend
  action: ALLOW
  rules:
  - from:
    - source:
        principals: ["cluster.local/ns/my-app/sa/backend"]
```

### Enforce strict mTLS:
```yaml
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: my-app
spec:
  mtls:
    mode: STRICT
```

## 🎪 Demo Application

Deploy Istio's sample bookinfo app in ambient mode:

```bash
# Create namespace
kubectl create namespace bookinfo

# Enable ambient mode
kubectl label namespace bookinfo istio.io/dataplane-mode=ambient

# Deploy app (no sidecars!)
kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-1.21/samples/bookinfo/platform/kube/bookinfo.yaml -n bookinfo

# Verify - should see 1/1 for all pods (not 2/2)
kubectl get pods -n bookinfo

# Access the app
kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-1.21/samples/bookinfo/networking/bookinfo-gateway.yaml -n bookinfo

# Get ingress URL
kubectl get svc -n istio-system istio-ingressgateway
```

## 🔄 Migration from Sidecar to Ambient

If you have existing Istio with sidecars:

```bash
# 1. Deploy ambient mode components (already done by playbook)

# 2. For each namespace, gradually switch:
# Remove sidecar injection label
kubectl label namespace my-app istio-injection-

# Add ambient mode label
kubectl label namespace my-app istio.io/dataplane-mode=ambient

# 3. Restart pods to remove sidecars
kubectl rollout restart deployment -n my-app

# 4. Verify sidecars are gone
kubectl get pods -n my-app
# Should show 1/1, not 2/2
```

## 📈 Resource Comparison

Check resource usage difference:

```bash
# Before (with sidecars): Each pod uses ~50-100MB extra for Envoy
kubectl top pods -n my-app-with-sidecars

# After (ambient mode): Zero overhead in pods
kubectl top pods -n my-app-ambient

# ztunnel uses ~100-200MB per node (shared across ALL pods)
kubectl top pods -n istio-system -l app=ztunnel
```

## 🎓 Key Concepts

**ztunnel** (Zero Trust Tunnel):
- Runs as DaemonSet (one per node)
- Intercepts traffic using eBPF
- Provides L4 features (mTLS, L4 authz, telemetry)
- Shared by all pods on the node

**Waypoint Proxy** (optional):
- Per-namespace or per-service L7 proxy
- Only needed for advanced routing
- Still more efficient than per-pod sidecars
- Deploy on-demand

**Enrollment Model**:
- Namespace-level: Label namespace
- Pod-level: Label individual pods
- Workload-level: Use ServiceEntry for VMs

## 🔍 Verification Commands

```bash
# Check ambient mode is enabled
istioctl experimental waypoint list

# Verify ztunnel is healthy
kubectl get pods -n istio-system -l app=ztunnel
kubectl logs -n istio-system -l app=ztunnel --tail=50

# Check which namespaces are in ambient mode
kubectl get namespaces -l istio.io/dataplane-mode=ambient

# Verify mTLS is working
istioctl proxy-config secret -n istio-system <ztunnel-pod-name>

# Check policy status
kubectl get authorizationpolicies -A
kubectl get peerauthentications -A
```

## 🚫 Disable Ambient Mode

To remove a namespace from the mesh:

```bash
kubectl label namespace my-app istio.io/dataplane-mode-
```

Pods continue running unchanged - no restart needed!

## 📚 Learn More

- [Istio Ambient Mode Docs](https://istio.io/latest/docs/ambient/)
- [Getting Started Guide](https://istio.io/latest/docs/ambient/getting-started/)
- [Architecture Deep Dive](https://istio.io/latest/docs/ambient/architecture/)
