# RKE2 Kubernetes Cluster Setup with Ansible

This playbook automates the installation and configuration of RKE2 (Rancher Kubernetes Engine 2) on EC2 Ubuntu instances.

## Architecture

- **Control Plane (Server)**: First host in inventory - runs Kubernetes API and etcd
- **Worker Nodes (Agents)**: Remaining hosts - run workloads

## Prerequisites

1. **EC2 Instances**:
   - Ubuntu 20.04 LTS or later (22.04 LTS recommended)
   - At least 2 vCPU and 4GB RAM per node
   - Security group allowing:
     - Port 6443 (Kubernetes API) between nodes
     - Port 10250 (kubelet) between nodes
     - Outbound HTTPS (443) for package downloads

2. **Local Setup**:
   - Ansible installed (`pip install ansible`)
   - SSH key file for EC2 instances
   - Network connectivity to instances

3. **EC2 Instances Running**:
   - Instances must be in running state
   - Private IPs must be accessible from your Ansible control machine

## Quick Start

### 1. Update Inventory

Edit `inventory-rke2.ini`:

```ini
[rke2_cluster]
rke2-control-1 ansible_host=10.0.1.10 ansible_user=ubuntu ansible_ssh_private_key_file=~/.ssh/your-key.pem
rke2-worker-1 ansible_host=10.0.1.11 ansible_user=ubuntu ansible_ssh_private_key_file=~/.ssh/your-key.pem
rke2-worker-2 ansible_host=10.0.1.12 ansible_user=ubuntu ansible_ssh_private_key_file=~/.ssh/your-key.pem

[rke2_cluster:vars]
rke2_token=your-secret-token-here
```

**Key points**:
- Replace IPs with your EC2 private IPs
- Replace `your-key.pem` with your actual SSH key path
- `rke2_token`: Use a strong, random token (min 32 chars) - keeps worker nodes secure
- Number of nodes can be any count (add/remove worker lines as needed)

### 2. Scaling Nodes

To add more worker nodes:

```ini
rke2-worker-3 ansible_host=10.0.1.13 ansible_user=ubuntu ansible_ssh_private_key_file=~/.ssh/your-key.pem
rke2-worker-4 ansible_host=10.0.1.14 ansible_user=ubuntu ansible_ssh_private_key_file=~/.ssh/your-key.pem
```

Then run the playbook again (only new nodes will be configured).

### 3. Run the Playbook

```bash
# Syntax check
ansible-playbook -i inventory-rke2.ini rke2-cluster.yml --syntax-check

# Dry run (no changes)
ansible-playbook -i inventory-rke2.ini rke2-cluster.yml --check

# Execute
ansible-playbook -i inventory-rke2.ini rke2-cluster.yml -v
```

### 4. Access Your Cluster

After successful deployment:

```bash
# Kubeconfig will be fetched to current directory
cat kubeconfig-rke2

# Replace localhost with control plane IP
export KUBECONFIG=./kubeconfig-rke2
kubectl get nodes
kubectl get pods -A
```

## Configuration Options

Edit variables in `rke2-cluster.yml`:

### RKE2 Version
```yaml
rke2_version: "v1.28.0"  # or "latest"
rke2_channel: "stable"    # or "latest", "testing"
```

### Custom Config
Modify `/etc/rancher/rke2/config.yaml` sections to add:
- Additional Kubernetes arguments
- CNI plugin configuration
- Kubelet options
- Feature gates

Example to add:
```yaml
- name: Configure control plane (server)
  copy:
    dest: /etc/rancher/rke2/config.yaml
    content: |
      server: https://{{ control_plane_ip }}:6443
      token: {{ rke2_token }}
      etcd-arg:
        - auto-compaction-retention=72h
      kubelet-arg:
        - max-pods=200
```

## Verification

Check cluster health:

```bash
# Nodes status
kubectl get nodes -o wide

# Core services
kubectl get pods -n kube-system

# Resources
kubectl top nodes
kubectl top pods -A
```

## Troubleshooting

### Nodes not joining
1. Check SSH connectivity: `ssh -i key.pem ubuntu@IP`
2. Verify token: Same token on all nodes
3. Check security group: Ports 6443, 10250 open
4. Review logs: `ssh ubuntu@IP "sudo journalctl -u rke2-agent -f"`

### Control plane not starting
```bash
# SSH to control node
ssh -i key.pem ubuntu@IP
sudo journalctl -u rke2-server -f
sudo /var/lib/rancher/rke2/bin/kubectl --kubeconfig /etc/rancher/rke2/rke2.yaml get nodes
```

### High latency between nodes
- Check VPC security groups
- Verify instance placement group (for performance)
- Consider instance type (network-optimized instances preferred)

## Security Notes

- **Token**: Keep `rke2_token` secret - treat like a password
- **SSH Keys**: Ensure SSH keys have restrictive permissions (chmod 600)
- **Security Groups**: Only open necessary ports
- **Updates**: Regularly update RKE2 by changing `rke2_version`

## Production Considerations

For production deployments:

1. **HA Control Plane**: Deploy 3+ control planes (requires Terraform/load balancer)
2. **Networking**: Use CNI like Cilium or Flannel
3. **Storage**: Configure persistent volume provisioners (Local, EBS, etc.)
4. **Monitoring**: Deploy Prometheus/Grafana or similar
5. **Logging**: Setup ELK or Loki for log aggregation
6. **Backup**: Configure etcd backups
7. **Network Policy**: Implement Kubernetes network policies

## Scaling Up/Down

### Add Worker Nodes
1. Launch new EC2 instances
2. Add to inventory
3. Run playbook again

### Remove Worker Nodes
1. Cordon and drain: `kubectl drain node-name --ignore-daemonsets`
2. Delete from cluster: `kubectl delete node node-name`
3. Remove from inventory
4. Stop EC2 instance

## Additional Resources

- [RKE2 Official Docs](https://docs.rke2.io/)
- [Rancher Documentation](https://rancher.com/docs/)
- [Kubernetes Documentation](https://kubernetes.io/docs/)
