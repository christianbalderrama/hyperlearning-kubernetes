# RHEL 10 Specific Configuration Guide

## 🎯 Why RHEL 10 for Kubernetes?

**Enterprise-Grade Benefits:**
- ✅ **Long-term support**: 10+ years of security updates
- ✅ **Stability**: Tested and certified for mission-critical workloads
- ✅ **Security**: SELinux, FIPS compliance, hardened by default
- ✅ **Support**: Official Red Hat support available
- ✅ **Compliance**: Meets regulatory requirements (HIPAA, PCI-DSS, etc.)

**The Trade-off:**
- More conservative package versions (stability over bleeding edge)
- SELinux can be tricky (but adds security)
- Subscription model for official repos (we use CentOS Stream for Docker)

## 🔧 RHEL vs Ubuntu Differences in This Playbook

### Package Manager
**Ubuntu**: `apt` (Advanced Package Tool)
**RHEL**: `dnf` (Dandified YUM)

```bash
# Ubuntu way
apt install containerd.io

# RHEL way
dnf install containerd.io
```

### SELinux (Security-Enhanced Linux)
**Key Concept**: SELinux is like a strict security guard that enforces mandatory access control.

**The Problem**: Kubernetes and SELinux don't play well together (yet).

**Our Solution**: Set SELinux to permissive mode
```bash
# Check status
getenforce

# Permissive = logs violations but doesn't block
# Enforcing = blocks violations
# Disabled = completely off (not recommended)
```

**Why permissive instead of disabled?**
- Still logs security violations (useful for auditing)
- Can potentially re-enable in future with proper policies
- Maintains some security context

**Production Note**: Some enterprises require SELinux enforcing. Solutions:
1. Use CRI-O instead of containerd (better SELinux support)
2. Apply custom SELinux policies for Kubernetes
3. Use Red Hat's supported solutions (OpenShift)

### Firewall
**Ubuntu**: `ufw` (Uncomplicated Firewall)
**RHEL**: `firewalld` (Dynamic firewall daemon)

**Our playbook opens these ports:**
```
6443/tcp      → API server (kubectl talks here)
2379-2380/tcp → etcd (cluster data store)
10250/tcp     → kubelet API (node agent)
10251/tcp     → kube-scheduler
10252/tcp     → kube-controller-manager
10255/tcp     → kubelet read-only API
30000-32767/tcp → NodePort services (expose apps)
```

**Verify it worked:**
```bash
firewall-cmd --list-all
```

### Repository Setup
**Docker CE Repository**: Uses CentOS Stream repo (compatible with RHEL)
**Kubernetes Repository**: Official K8s RPM repository

**Why not use RHEL's built-in repos?**
- RHEL repos may have older versions
- Official K8s repos give us latest stable releases
- Docker CE provides better containerd integration

## 🔒 Security Considerations on RHEL

### 1. SELinux Contexts
Even in permissive mode, SELinux logs potential violations:
```bash
# View SELinux denials
ausearch -m avc -ts recent

# Common K8s-related denials are expected and safe in permissive mode
```

### 2. Firewall Zones
RHEL uses firewalld zones for network segmentation:
```bash
# Check current zone
firewall-cmd --get-default-zone

# Our playbook adds rules to the default zone
# For multi-NIC setups, consider:
firewall-cmd --zone=public --add-port=6443/tcp --permanent
firewall-cmd --zone=internal --add-source=10.244.0.0/16 --permanent
```

### 3. Container Registry Access
If using private registries behind corporate firewall:
```bash
# Configure containerd proxy
mkdir -p /etc/systemd/system/containerd.service.d
cat > /etc/systemd/system/containerd.service.d/http-proxy.conf <<EOF
[Service]
Environment="HTTP_PROXY=http://proxy.example.com:8080"
Environment="HTTPS_PROXY=http://proxy.example.com:8080"
Environment="NO_PROXY=localhost,127.0.0.1,10.0.0.0/8,172.16.0.0/12,192.168.0.0/16"
EOF
systemctl daemon-reload
systemctl restart containerd
```

## 📦 Package Version Management

Unlike Ubuntu's `apt-mark hold`, RHEL uses different approach:

**Lock packages to prevent updates:**
```bash
# Add to /etc/yum.conf or /etc/dnf/dnf.conf
exclude=kubelet kubeadm kubectl

# Or use versionlock plugin
dnf install python3-dnf-plugin-versionlock
dnf versionlock kubelet kubeadm kubectl

# List locked versions
dnf versionlock list
```

**Why version locking matters:**
- Kubernetes components must be compatible versions
- Accidental `dnf update` could break cluster
- Upgrades should be intentional and tested

## 🔄 Subscription Manager

**Do I need RHEL subscription?**

For this playbook: **No** for basic setup
- We use Docker CE repo (doesn't require RHEL subscription)
- We use upstream Kubernetes repo (free)

For production: **Yes, recommended**
- Access to RHEL-tested packages
- Security updates and CVE fixes
- Support from Red Hat
- Compliance requirements

**If you have a subscription:**
```bash
# Register system
subscription-manager register --username=<username> --password=<password>

# Attach subscription
subscription-manager attach --auto

# Enable required repos
subscription-manager repos --enable=rhel-10-for-x86_64-baseos-rpms
subscription-manager repos --enable=rhel-10-for-x86_64-appstream-rpms
```

## 🚀 Performance Tuning for RHEL

### 1. Tuned Profile
RHEL includes `tuned` for system optimization:
```bash
# Install tuned
dnf install tuned

# Set throughput-performance profile (good for K8s)
tuned-adm profile throughput-performance

# Verify active profile
tuned-adm active
```

### 2. Transparent Huge Pages
Disable for better container performance:
```bash
# Check current setting
cat /sys/kernel/mm/transparent_hugepage/enabled

# Disable (add to /etc/rc.local)
echo never > /sys/kernel/mm/transparent_hugepage/enabled
echo never > /sys/kernel/mm/transparent_hugepage/defrag
```

### 3. System Limits
Increase for container workloads:
```bash
# Edit /etc/security/limits.conf
* soft nofile 65536
* hard nofile 65536
* soft nproc 65536
* hard nproc 65536

# Edit /etc/sysctl.conf
fs.inotify.max_user_instances=8192
fs.inotify.max_user_watches=524288
```

## 🔍 Monitoring RHEL-Specific Metrics

### System Resources
```bash
# CPU usage
mpstat -P ALL 1

# Memory detailed
free -h
cat /proc/meminfo

# Disk I/O
iostat -x 1

# Network
sar -n DEV 1
```

### Journal Logs
RHEL uses systemd journal extensively:
```bash
# View kubelet logs
journalctl -u kubelet -f

# View containerd logs
journalctl -u containerd -f

# View all K8s related services
journalctl -u kube* -f

# Persistent logs (survive reboots)
mkdir -p /var/log/journal
systemctl restart systemd-journald
```

## 📊 Compliance and Hardening

### OpenSCAP
RHEL includes security compliance tools:
```bash
# Install OpenSCAP
dnf install openscap-scanner scap-security-guide

# Scan for compliance (NIST 800-53, PCI-DSS, etc.)
oscap xccdf eval --profile pci-dss \
  --results /tmp/scan-results.xml \
  /usr/share/xml/scap/ssg/content/ssg-rhel10-ds.xml
```

### Hardening for Production
```bash
# 1. Disable unnecessary services
systemctl disable postfix
systemctl disable cups

# 2. Configure auditd for K8s events
auditctl -w /etc/kubernetes/ -p wa -k k8s-config-changes
auditctl -w /var/lib/kubelet/ -p wa -k kubelet-changes

# 3. Set up AIDE (file integrity monitoring)
dnf install aide
aide --init
mv /var/lib/aide/aide.db.new.gz /var/lib/aide/aide.db.gz
aide --check
```

## 🌐 Networking on RHEL

### NetworkManager vs network-scripts
RHEL 10 uses NetworkManager by default:
```bash
# Check NetworkManager status
systemctl status NetworkManager

# Manage connections
nmcli connection show
nmcli device status

# For K8s, ensure interfaces are managed
nmcli connection modify eth0 connection.autoconnect yes
```

### IPv6 Considerations
If not using IPv6, disable to avoid routing issues:
```bash
# Edit /etc/sysctl.conf
net.ipv6.conf.all.disable_ipv6 = 1
net.ipv6.conf.default.disable_ipv6 = 1

# Apply
sysctl -p
```

## 🎓 Migration from Ubuntu to RHEL

**Command Equivalents:**

| Task | Ubuntu | RHEL 10 |
|------|--------|---------|
| Install package | `apt install pkg` | `dnf install pkg` |
| Update packages | `apt update && apt upgrade` | `dnf update` |
| Search package | `apt search pkg` | `dnf search pkg` |
| Remove package | `apt remove pkg` | `dnf remove pkg` |
| List installed | `dpkg -l` | `rpm -qa` |
| Package info | `apt show pkg` | `dnf info pkg` |
| Clean cache | `apt clean` | `dnf clean all` |
| Firewall | `ufw allow 80/tcp` | `firewall-cmd --add-port=80/tcp` |

## 📚 Additional Resources

- [RHEL 10 Documentation](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/10)
- [Kubernetes on RHEL Best Practices](https://access.redhat.com/articles/4324501)
- [SELinux User Guide](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/10/html/using_selinux/)
- [Firewalld Documentation](https://firewalld.org/documentation/)

## 🔐 Enterprise Support Options

1. **Red Hat Subscription**: Official support, certified packages
2. **Red Hat OpenShift**: Enterprise K8s platform (overkill for basic needs)
3. **Community Support**: CentOS Stream, Kubernetes Slack, Reddit

**Our Playbook Philosophy**: Enterprise-grade foundation with community components
- RHEL 10 = Stable OS foundation
- Upstream K8s = Latest features
- Istio Ambient = Cutting-edge service mesh
- You get: Stability + Innovation
