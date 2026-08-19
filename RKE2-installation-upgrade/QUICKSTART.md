# RKE2 Ansible Playbook - Quick Start Guide

## What You Have

A role-based Ansible structure to install and manage RKE2 (Rancher Kubernetes) on EC2 Ubuntu instances.

### Directory Layout
```
roles/
├── rke2-common/      # Install RKE2 binaries & dependencies (all nodes)
├── rke2-server/      # Setup control plane (first node)
├── rke2-agent/       # Setup worker nodes (remaining nodes)
└── rke2-upgrade/     # Upgrade RKE2 version (future use)

group_vars/rke2_cluster.yml   # Shared config for all nodes
inventory.ini                  # Your EC2 nodes definition
rke2-install.yml              # Main installation playbook
rke2-upgrade.yml              # Upgrade playbook (ready to use)
```

## 1. Configure Your Nodes

Edit `inventory.ini` with your EC2 instance details:

```ini
[rke2_cluster]
rke2-control-1 ansible_host=10.0.1.10 ansible_user=ubuntu ansible_ssh_private_key_file=~/.ssh/your-key.pem
rke2-worker-1 ansible_host=10.0.1.11 ansible_user=ubuntu ansible_ssh_private_key_file=~/.ssh/your-key.pem
rke2-worker-2 ansible_host=10.0.1.12 ansible_user=ubuntu ansible_ssh_private_key_file=~/.ssh/your-key.pem
```

Key points:
- **First host** becomes control plane (server)
- **Rest** become worker nodes (agents)
- Replace IP addresses with your EC2 private IPs
- Replace SSH key path with yours

## 2. Set RKE2 Token

Edit `group_vars/rke2_cluster.yml`:

```yaml
rke2_token: "your-secure-32-character-token-here"
```

Generate a secure token:
```bash
head -c 32 /dev/urandom | base64
```

## 3. Install Cluster

```bash
# Check playbook syntax
ansible-playbook -i inventory.ini rke2-install.yml --syntax-check

# Dry run (safe, no changes)
ansible-playbook -i inventory.ini rke2-install.yml --check

# Execute installation
ansible-playbook -i inventory.ini rke2-install.yml -v
```

Time to complete: ~5-10 minutes depending on network

## 4. Access Your Cluster

After playbook completes, kubectl is installed on the control plane node.

SSH to control plane and run kubectl commands:

```bash
# SSH to control plane
ssh -i ~/github/ec2.pem ubuntu@52.201.216.243

# kubectl is ready to use (kubeconfig already configured)
kubectl get nodes -o wide

# Check system pods
kubectl get pods -n kube-system

# Monitor cluster
kubectl top nodes

# Or use with explicit kubeconfig
export KUBECONFIG=/etc/rancher/rke2/rke2.yaml
kubectl get nodes
```

## 5. Scale the Cluster

### Add Worker Nodes

```bash
# 1. Launch new EC2 instance
# 2. Add to inventory.ini
rke2-worker-3 ansible_host=10.0.1.13 ansible_user=ubuntu ansible_ssh_private_key_file=~/.ssh/your-key.pem

# 3. Run playbook (only on new node)
ansible-playbook -i inventory.ini rke2-install.yml --limit rke2-worker-3 -v

# 4. Verify
kubectl get nodes
```

### Remove Worker Nodes

```bash
# 1. Drain workloads
kubectl drain rke2-worker-1 --ignore-daemonsets

# 2. Delete from cluster
kubectl delete node rke2-worker-1

# 3. Remove from inventory.ini
# 4. Terminate EC2 instance
```

## 6. Upgrade RKE2 (Future Use)

When you need to upgrade to a new RKE2 version:

```bash
# Edit group_vars/rke2_cluster.yml
rke2_version: "v1.29.0"

# Or pass on command line
ansible-playbook -i inventory.ini rke2-upgrade.yml -e "rke2_upgrade_version=v1.29.0" -v
```

## Advanced: Run Specific Roles

```bash
# Only run on specific nodes
ansible-playbook -i inventory.ini rke2-install.yml --limit rke2-control-1

# Only run specific role
ansible-playbook -i inventory.ini rke2-install.yml -t rke2-common

# Only run server setup
ansible-playbook -i inventory.ini rke2-install.yml -t rke2-server

# Only run agent setup
ansible-playbook -i inventory.ini rke2-install.yml -t rke2-agent
```

## Customization Examples

### Add Node Labels

Create `host_vars/rke2-worker-1.yml`:

```yaml
rke2_agent_config:
  node-label:
    - workload=compute
    - zone=us-east-1a
```

### Custom Kubelet Settings

Edit `group_vars/rke2_cluster.yml`:

```yaml
rke2_server_config:
  disable:
    - local-storage
  kubelet_arg:
    - max-pods=200
    - kube-reserved=cpu=100m,memory=128Mi

rke2_agent_config:
  kubelet_arg:
    - max-pods=150
```

## Troubleshooting

### Nodes not joining
```bash
# Check SSH connectivity
ssh -i ~/.ssh/your-key.pem ubuntu@10.0.1.11

# Check agent logs
ssh -i ~/.ssh/your-key.pem ubuntu@10.0.1.11 sudo journalctl -u rke2-agent -f

# Verify token matches in group_vars
```

### Control plane not starting
```bash
# SSH to control node
ssh -i ~/.ssh/your-key.pem ubuntu@10.0.1.10

# Check server logs
sudo journalctl -u rke2-server -f

# Verify node token
sudo cat /etc/rancher/rke2/node-token
```

### Check role execution
```bash
# Show all tasks that would run
ansible-playbook -i inventory.ini rke2-install.yml --check -vv

# Debug variables
ansible -i inventory.ini rke2_cluster -m debug -a "var=rke2_token"
```

## Security Checklist

- [ ] Changed `rke2_token` from default in `group_vars/rke2_cluster.yml`
- [ ] SSH key file has restricted permissions: `chmod 600 ~/.ssh/your-key.pem`
- [ ] EC2 security group allows:
  - Port 6443 (Kubernetes API) between nodes
  - Port 10250 (kubelet) between nodes
- [ ] Removed default SSH access (use ansible user only)
- [ ] Stored kubeconfig securely: `chmod 600 kubeconfig-rke2`

## What Each Role Does

| Role | When | What |
|------|------|------|
| rke2-common | All nodes | Install RKE2 binaries, dependencies, create directories |
| rke2-server | First node only | Setup control plane, start API server |
| rke2-agent | Worker nodes | Join cluster as worker nodes |
| rke2-upgrade | All nodes (serial) | Upgrade RKE2 version with zero-downtime |

## Next Steps

1. **Post-install**: Deploy ingress controller, storage provisioner, monitoring
2. **Production**: Add 2 more control planes for HA, setup etcd backup
3. **Monitoring**: Deploy Prometheus/Grafana for cluster observability
4. **Logging**: Setup Loki or ELK for log aggregation

See `ROLES-STRUCTURE.md` for detailed documentation.
