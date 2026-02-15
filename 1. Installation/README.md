# Kubernetes Installation with Istio Ambient Mode - Production Grade (RHEL 10)

## 🎯 The Analogy
Think of this setup like building a smart city on **enterprise-grade infrastructure (RHEL)**:
- **Base CNI (Cilium)** = Roads and highways (basic pod-to-pod communication)
- **Istio Ambient Mode** = Smart traffic management system (security, observability, routing) WITHOUT installing sensors in every car
- **ztunnel** = Invisible security checkpoints at every intersection (L4 zero-trust)
- **No Sidecars** = Your applications stay lightweight - no extra containers!
- **RHEL 10** = The bedrock foundation (enterprise stability and support)

Traditional Istio = putting a GPS tracker in every car (sidecar)
Ambient Mode = city-wide traffic camera system (shared infrastructure)

## 🚀 Quick Start

### 1. Edit the inventory
```bash
vi inventory.ini
# Replace 192.168.1.10 with your master node IP
```

### 2. Run the playbook
```bash
ansible-playbook -i inventory.ini k8s-kubeadm-install.yml
```

### 3. Watch it work!
The playbook will:
- ✅ Set up all prerequisites
- ✅ Install containerd (container runtime)
- ✅ Install Kubernetes components
- ✅ Initialize the cluster
- ✅ Install Cilium CNI (eBPF-based networking)
- ✅ Install Istio in ambient mode
- ✅ Run comprehensive verification tests
- ✅ Deploy test pod with ambient mesh enabled

## 📋 What Each Step Does (Feynman Style)

### Step 1: Prerequisites
**Why?** Kubernetes is picky - it won't run with swap enabled (like trying to drive with the parking brake on). We also need special kernel settings so container networks can talk to each other.

### Step 2: Container Runtime
**Why?** Kubernetes doesn't run containers itself - it needs containerd to actually run them. Think of K8s as the manager and containerd as the worker.

### Step 3: Install K8s Components
- **kubelet**: The agent on each node (like a factory floor supervisor)
- **kubeadm**: Bootstrap tool (the construction foreman)
- **kubectl**: Command-line tool (your walkie-talkie to talk to the cluster)

### Step 4: Initialize Control Plane
**Why?** This creates the "brain" of your cluster - the API server, scheduler, and controller manager that make all decisions.

### Step 5: Base CNI (Cilium)
**Why?** Pods need to communicate. Cilium uses eBPF (like reprogramming the kernel itself) for ultra-fast networking. It's the foundation highway system.

**Key insight**: You NEED a base CNI. Istio ambient mode is NOT a CNI - it's a service mesh that layers on top!

### Step 6: Istio Ambient Mode (Revolutionary!)
**Why ambient mode?**
- **Traditional Istio**: Injects a sidecar proxy into EVERY pod (like adding a security guard to every room)
- **Ambient Mode**: Shared infrastructure handles everything (like building security cameras covering all areas)

**How it works:**
1. **ztunnel** (zero-trust tunnel): DaemonSet on each node that intercepts traffic at L4
2. **No sidecars**: Your app pods stay clean and lightweight
3. **Namespace-level enablement**: Just label a namespace: `istio.io/dataplane-mode=ambient`

**Benefits:**
- 🚀 **Lower resource usage**: No sidecar = less CPU/memory per pod
- 🔧 **Simpler operations**: No need to restart pods to update mesh
- 🎯 **Gradual adoption**: Enable per namespace, not per pod
- 🔒 **Same security**: mTLS, authorization policies, telemetry all work

### Step 7: Verification
**Why?** Trust but verify! We:
1. Check nodes are Ready
2. Check all system pods are Running
3. Verify ztunnel daemonset is deployed
4. Deploy test pod in ambient-enabled namespace
5. Confirm mesh enrollment (no sidecars injected!)

## 🏗️ Architecture Diagram (Mental Model)

```
┌─────────────────────────────────────────────┐
│           Kubernetes Cluster                │
│                                             │
│  ┌─────────────────────────────────────┐   │
│  │  Pod (nginx-test)                   │   │
│  │  - No sidecar!                      │   │
│  │  - Just your app container          │   │
│  └─────────────────────────────────────┘   │
│           ↓ traffic                         │
│  ┌─────────────────────────────────────┐   │
│  │  ztunnel (DaemonSet on each node)   │   │
│  │  - Intercepts at L4                 │   │
│  │  - Enforces mTLS                    │   │
│  │  - Telemetry collection             │   │
│  └─────────────────────────────────────┘   │
│           ↓                                 │
│  ┌─────────────────────────────────────┐   │
│  │  Cilium CNI (eBPF kernel magic)     │   │
│  │  - Actual packet routing            │   │
│  │  - Network policies                 │   │
│  └─────────────────────────────────────┘   │
│           ↓                                 │
│        Physical Network                     │
└─────────────────────────────────────────────┘
```

## 🔍 Verification Output

After running, you'll see:
```
✅ Cluster is operational with Istio Ambient Mode!
✅ ztunnel daemonset deployed for L4 zero-trust networking
✅ No sidecars needed - ambient mode active!
```

Plus detailed output showing:
- Cluster info (API server endpoint)
- Node status (Ready/NotReady)
- All system pods (kube-system namespace)
- Istio components (istiod, ztunnel daemonset)
- Test nginx pod running in ambient mesh (no sidecar!)

**What to look for:**
```bash
# Check ztunnel is running on all nodes
kubectl get daemonset -n istio-system ztunnel

# Verify test pod has NO sidecar (should show 1/1, not 2/2)
kubectl get pod nginx-test -n test-ambient

# Confirm namespace is in ambient mode
kubectl get namespace test-ambient -o jsonpath='{.metadata.labels}'
# Should show: istio.io/dataplane-mode=ambient
```

## 🏗️ Production Considerations

This playbook includes:
- **SystemdCgroup**: Proper cgroup driver (critical for stability)
- **Cilium CNI**: Modern eBPF-based networking (faster than iptables)
- **Istio Ambient Mode**: Service mesh WITHOUT sidecars
  - **ztunnel**: Zero-trust L4 proxy (runs as DaemonSet)
  - **Namespace-level enablement**: Fine-grained control
  - **No pod restarts needed**: Just label the namespace
- **Version pinning**: Prevents accidental upgrades
- **Comprehensive verification**: Tests both cluster and mesh

## 🌟 Why Istio Ambient Mode?

**Traditional Service Mesh Problems:**
- Every pod gets a sidecar = 2x memory usage
- Sidecar updates require pod restarts
- Complex injection logic
- Harder to debug

**Ambient Mode Solutions:**
- Shared infrastructure (ztunnel per node, not per pod)
- Update mesh without touching apps
- Simpler: just label namespace
- Same security guarantees (mTLS, policies)

**When to use what:**
- **L4 only (most apps)**: Just enable ambient mode on namespace
- **L7 features needed**: Deploy waypoint proxy for specific services
- **No mesh needed**: Don't label the namespace

## 📊 Next Steps for Production

1. **Add monitoring**: Deploy Prometheus + Grafana + Kiali (for Istio visibility)
2. **Add logging**: Deploy ELK or Loki stack
3. **Storage**: Install CSI driver (Longhorn, Rook-Ceph)
4. **Ingress**: Use Istio Gateway (replaces nginx-ingress)
5. **Security**: 
   - Enable AuthorizationPolicies in Istio
   - Configure PeerAuthentication for strict mTLS
   - Set up Pod Security Standards

## 🎯 Using Ambient Mode

Enable ambient mesh for your apps:

```bash
# Label namespace to enable ambient mode
kubectl label namespace my-app istio.io/dataplane-mode=ambient

# Deploy your app normally (no sidecar injection!)
kubectl apply -f my-app.yaml -n my-app

# Traffic is now in the mesh with mTLS
# View metrics in Kiali
# Apply authorization policies
# All without sidecars!
```

For L7 features (header routing, retries, etc.):
```bash
# Deploy waypoint proxy for specific service
istioctl waypoint apply -n my-app --name my-service-waypoint
```

## 🔧 Troubleshooting

**RHEL-Specific Issues:**

**SELinux blocking pods?**
```bash
# Check SELinux status
getenforce
# Should show "Permissive"

# If showing "Enforcing", set to permissive
setenforce 0
sed -i 's/^SELINUX=enforcing/SELINUX=permissive/' /etc/selinux/config
```

**Firewall blocking traffic?**
```bash
# Check firewalld status
firewall-cmd --list-all

# Verify K8s ports are open
firewall-cmd --list-ports

# Manually open a port if needed
firewall-cmd --permanent --add-port=6443/tcp
firewall-cmd --reload
```

**DNF/repo issues?**
```bash
# Clear DNF cache
dnf clean all

# Check repo configuration
dnf repolist

# Verify Docker CE repo
cat /etc/yum.repos.d/docker-ce.repo
```

**Pods stuck in Pending?**
```bash
kubectl describe pod <pod-name> -n kube-system
```

**Node NotReady?**
```bash
kubectl describe node <node-name>
journalctl -u kubelet -f
```

**Cilium not working?**
```bash
cilium status
kubectl get pods -n kube-system | grep cilium
kubectl logs -n kube-system -l k8s-app=cilium
```

**Istio components not ready?**
```bash
# Check Istio installation status
istioctl verify-install

# Check istiod logs
kubectl logs -n istio-system -l app=istiod

# Check ztunnel logs
kubectl logs -n istio-system -l app=ztunnel
```

**Pod not in ambient mesh?**
```bash
# Verify namespace label
kubectl get namespace my-namespace -o jsonpath='{.metadata.labels}'

# Should show: istio.io/dataplane-mode=ambient

# Check if pod has sidecar (it shouldn't!)
kubectl get pod my-pod -n my-namespace -o jsonpath='{.spec.containers[*].name}'
# Should only show your app container, no istio-proxy

# Check ztunnel is capturing traffic
kubectl logs -n istio-system -l app=ztunnel | grep my-pod
```

**mTLS not working?**
```bash
# Check peer authentication
kubectl get peerauthentication -A

# Verify ztunnel is running
kubectl get daemonset -n istio-system ztunnel

# Test connection with mTLS
istioctl proxy-config secret -n istio-system <ztunnel-pod>
```

## 📝 Variables You Can Customize

In the playbook:
- `k8s_version`: "1.29" (change to desired version)
- `pod_network_cidr`: "10.244.0.0/16" (Cilium default, change if needed)
- `istio_version`: "1.21.0" (latest stable with ambient support)

**Important**: Keep pod_network_cidr consistent with your network design!

## 🔄 Traditional Istio vs Ambient Mode

**Traditional Istio (Sidecar Pattern):**
```
Pod: my-app
├── my-app container (your app)
└── istio-proxy container (Envoy sidecar)
    ├── Consumes CPU/memory per pod
    ├── Requires pod restart to update
    └── Injected automatically or manually
```

**Ambient Mode (Shared Infrastructure):**
```
Pod: my-app
└── my-app container (your app only!)

Node Infrastructure:
└── ztunnel (DaemonSet - one per node)
    ├── Handles all L4 traffic for ALL pods
    ├── Update without touching apps
    └── Lower total resource usage
```

**The Math:**
- 100 pods with sidecars: 100 Envoy proxies (200 containers total)
- 100 pods with ambient: 100 app containers + 1 ztunnel per node
- On 3-node cluster: 200 containers vs 103 containers!

## ⚠️ Prerequisites

- **RHEL 10** (tested and optimized)
- At least 2 CPU cores
- At least 2GB RAM (4GB recommended)
- SSH access with sudo privileges
- Ansible installed on control machine
- **SELinux**: Will be set to permissive mode (required for K8s)
- **Firewalld**: Will be configured automatically for K8s ports
- **RHEL Subscription**: Not required (uses CentOS Stream repos for Docker)
