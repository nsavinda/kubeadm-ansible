# Kubernetes Cluster Setup with Ansible

This Ansible playbook automates the deployment of a Kubernetes cluster with Cilium networking, ArgoCD, and Kubernetes Dashboard.

## What Gets Installed

- **Prerequisites**: System updates, Docker, UFW firewall rules, kernel modules
- **Kubernetes**: Control plane and worker nodes with kubeadm
- **Cilium**: eBPF-based networking with Hubble observability
- **ArgoCD**: GitOps continuous delivery tool
- **Kubernetes Dashboard**: Web-based UI for cluster management
- **Tools**: Helm package manager

## Prerequisites

- Ubuntu/Debian-based systems
- SSH access to all nodes
- Sudo privileges on all nodes
- Python 3 installed on all nodes

## Inventory Setup

Edit `inventory/hosts.ini`:

```ini
[control_plane]
cp1 ansible_host=<control_plane_ip>

[workers]
w1 ansible_host=<worker_ip1>
w2 ansible_host=<worker_ip2>

[all:vars]
ansible_user=<your_ssh_user>
ansible_become=yes
```

### Example:

```ini
[control_plane]
cp1 ansible_host=192.168.122.166

[workers]
w1 ansible_host=192.168.122.29
w2 ansible_host=192.168.122.146

[all:vars]
ansible_user=nirmal
ansible_become=yes
```

## Group Variables Configuration

Edit `group_vars/all.yml` to customize your deployment:

```yaml
# Kubernetes version
k8s_version: "1.30.0"

# Pod network CIDR for Cilium
pod_cidr: "10.244.0.0/16"

# Cilium version
cilium_version: "1.16.0"

# Control plane endpoint (uses first control plane node)
control_plane_endpoint: "{{ groups['control_plane'][0] }}"

# Optional: Override host IPs for /etc/hosts entries
# If not defined, uses ansible_host from inventory
host_ips:
  cp1: "192.168.122.166"
  w1: "192.168.122.29"
  w2: "192.168.122.146"

# Optional: ArgoCD configuration (uncomment to enable)
# argocd_version: "2.12.3"
# argocd_expose_type: "nodeport"
# argocd_nodeport_port: 30080
# argocd_grpc_nodeport_port: 30443
```

### Key Configuration Options:

- **`host_ips`**: Override IP addresses for `/etc/hosts` entries. If not defined, falls back to `ansible_host` from inventory.
- **`k8s_version`**: Kubernetes version to install
- **`cilium_version`**: Cilium CNI version
- **`pod_cidr`**: Pod network CIDR range

## How to Run

### 1. Prepare Your Environment

```bash
# Install Ansible (if not already installed)
pip install ansible

# Navigate to the ansible directory
cd /path/to/ansible

# Test connectivity to all nodes
ansible -i inventory/hosts.ini all -m ping
```

### 2. Run the Complete Playbook

```bash
ansible-playbook -i inventory/hosts.ini site.yml
```

### 3. Run Individual Roles (Optional)

```bash
# Run only prerequisites
ansible-playbook -i inventory/hosts.ini site.yml --tags prerequisites

# Run only Kubernetes setup
ansible-playbook -i inventory/hosts.ini site.yml --tags kubernetes

# Run only Cilium installation
ansible-playbook -i inventory/hosts.ini site.yml --tags cilium
```


## Firewall Rules

The playbook configures UFW with the following rules:
- SSH (22)
- Kubernetes API (6443) - control plane only
- etcd (2379-2380) - control plane only
- Kubelet (10250)
- NodePort services (30000-32767)
- Cilium VXLAN (8472/udp)
- Cilium health checks (4240/tcp)
- Hubble (4244/tcp, 4245/tcp)

## Troubleshooting

### Check cluster status
```bash
kubectl get nodes
kubectl get pods -A
```

### Check Cilium status
```bash
cilium status
cilium connectivity test
```

### View logs
```bash
# Control plane logs
journalctl -u kubelet -f

# Cilium logs
kubectl logs -n kube-system -l k8s-app=cilium
```

## Directory Structure

```
ansible/
├── inventory/
│   └── hosts.ini          # Ansible inventory
├── group_vars/
│   └── all.yml           # Global variables
├── roles/
│   ├── prerequisites/    # System preparation
│   ├── kube-components/  # Kubernetes components
│   ├── control-plane/    # Control plane setup
│   ├── workers/          # Worker node setup
│   ├── cilium/           # CNI installation
│   ├── tools/            # Additional tools (ArgoCD, Dashboard)
│   └── apps/             # Application deployments
└── site.yml              # Main playbook
```