# RKE2 Ansible Roles Structure

This directory follows a role-based Ansible structure for better organization, reusability, and maintainability.

## Directory Structure

```
ansible-demo/
├── rke2-install.yml              # Main playbook for installation
├── rke2-upgrade.yml              # Playbook for upgrades (future)
├── inventory.ini                 # Inventory file with nodes
├── group_vars/
│   └── rke2_cluster.yml          # Shared variables for all nodes
├── host_vars/                    # Optional: per-host variables
├── roles/
│   ├── rke2-common/              # Common tasks for all nodes
│   │   ├── defaults/main.yml
│   │   └── tasks/main.yml
│   ├── rke2-server/              # Control plane specific tasks
│   │   ├── defaults/main.yml
│   │   ├── tasks/main.yml
│   │   ├── handlers/main.yml
│   │   └── templates/config.yaml.j2
│   ├── rke2-agent/               # Worker node specific tasks
│   │   ├── defaults/main.yml
│   │   ├── tasks/main.yml
│   │   ├── handlers/main.yml
│   │   └── templates/config.yaml.j2
│   └── rke2-upgrade/             # Upgrade tasks (future)
│       └── ...
└── ROLES-STRUCTURE.md            # This file
```

## Roles Explained

### rke2-common
**Purpose**: Installation and setup common to all nodes

**Tasks**:
- Update system packages
- Install dependencies (curl, git, jq, etc.)
- Download RKE2 binaries
- Create directory structure
- Set hostname
- Create kubectl/crictl symlinks
- Verify installation

**Variables** (in `defaults/main.yml`):
- `rke2_version`: RKE2 release version
- `rke2_channel`: Release channel (stable, latest, testing)
- `rke2_config_dir`: Config directory path
- `rke2_data_dir`: Data directory path
- `rke2_token`: Cluster authentication token

### rke2-server
**Purpose**: Control plane/server specific setup

**Tasks**:
- Generate server config file from template
- Create systemd service links
- Start rke2-server service
- Wait for control plane readiness
- Extract and store node token

**Handlers**:
- Restart RKE2 server
- Reload systemd

**Template** (`config.yaml.j2`):
- Server configuration with token
- Bind and advertise addresses
- ETCD arguments
- Kubelet arguments
- Disabled components

**When to use**: Only on the first host in inventory (control plane)

### rke2-agent
**Purpose**: Worker node specific setup

**Tasks**:
- Wait for control plane to be ready
- Generate agent config file from template
- Create systemd service links
- Start rke2-agent service
- Wait for agent readiness

**Handlers**:
- Restart RKE2 agent
- Reload systemd

**Template** (`config.yaml.j2`):
- Server URL
- Agent token
- Kubelet arguments
- Node labels

**When to use**: On all hosts except the first (worker nodes)

## Variable Hierarchy

Ansible variables are loaded in this order (later overrides earlier):

1. **defaults/main.yml** - Hardcoded defaults in each role
2. **group_vars/rke2_cluster.yml** - Variables for all nodes in group
3. **host_vars/hostname.yml** - Variables for specific host (if created)
4. **-e extra-vars** - Command line variables

### Example Override

Override token via command line:
```bash
ansible-playbook -i inventory.ini rke2-install.yml -e "rke2_token=your-new-token"
```

Override per-host:
```yaml
# host_vars/rke2-worker-1.yml
rke2_agent_config:
  kubelet-arg:
    - max-pods=150
    - kube-reserved=cpu=100m,memory=128Mi
```

## Running Playbooks

### Installation
```bash
# Syntax check
ansible-playbook -i inventory.ini rke2-install.yml --syntax-check

# Dry run
ansible-playbook -i inventory.ini rke2-install.yml --check

# Execute
ansible-playbook -i inventory.ini rke2-install.yml -v
```

### Run specific role
```bash
# Only run common tasks
ansible-playbook -i inventory.ini rke2-install.yml -t rke2-common

# Only run server tasks
ansible-playbook -i inventory.ini rke2-install.yml -t rke2-server

# Only run agent tasks
ansible-playbook -i inventory.ini rke2-install.yml -t rke2-agent
```

## Configuration Examples

### Custom Kubelet Arguments

Edit `group_vars/rke2_cluster.yml`:

```yaml
rke2_server_config:
  disable:
    - local-storage
  kubelet_arg:
    - max-pods=200
    - kube-reserved=cpu=100m,memory=128Mi
    - system-reserved=cpu=50m,memory=64Mi

rke2_agent_config:
  kubelet_arg:
    - max-pods=150
    - pod-infra-container-image=...
```

### Custom CNI (Flannel VXLAN)

Add to `group_vars/rke2_cluster.yml`:

```yaml
rke2_server_config:
  disable:
    - local-storage
  flannel_backend: vxlan
```

### Node Labels

Edit `host_vars/rke2-worker-1.yml`:

```yaml
rke2_agent_config:
  node-label:
    - workload=compute
    - zone=us-east-1a
```

## Future: rke2-upgrade Role

For upgrading RKE2 versions, create `roles/rke2-upgrade/`:

```yaml
# roles/rke2-upgrade/tasks/main.yml
- name: Drain node before upgrade
  shell: kubectl drain {{ inventory_hostname }} --ignore-daemonsets

- name: Update RKE2 version
  shell: curl -sfL https://get.rke2.io | INSTALL_RKE2_VERSION="{{ new_rke2_version }}" sh -

- name: Restart service
  systemd:
    name: "{{ rke2_service_name }}"
    state: restarted

- name: Uncordon node
  shell: kubectl uncordon {{ inventory_hostname }}
```

Playbook:
```yaml
# rke2-upgrade.yml
---
- name: Upgrade RKE2 Cluster
  hosts: rke2_cluster
  serial: 1  # Upgrade one node at a time
  become: yes
  
  pre_tasks:
    - name: Set new version
      set_fact:
        new_rke2_version: "v1.28.0"
  
  roles:
    - role: rke2-upgrade
```

## Scaling the Cluster

### Add Worker Nodes

1. Launch new EC2 instance
2. Add to `inventory.ini`:
```ini
rke2-worker-3 ansible_host=10.0.1.13 ansible_user=ubuntu
```

3. Run playbook:
```bash
ansible-playbook -i inventory.ini rke2-install.yml --limit rke2-worker-3
```

### Remove Worker Nodes

1. Drain the node:
```bash
kubectl drain rke2-worker-1 --ignore-daemonsets
```

2. Delete from cluster:
```bash
kubectl delete node rke2-worker-1
```

3. Remove from `inventory.ini`
4. Terminate EC2 instance

## Troubleshooting

### Check role execution
```bash
# Show what tasks would run for each host
ansible-playbook -i inventory.ini rke2-install.yml --check -vv
```

### Check variable values
```bash
# Debug variables
ansible -i inventory.ini rke2_cluster -m debug -a "var=rke2_token"
```

### Check logs
```bash
# SSH to node and check service
ssh ubuntu@host sudo journalctl -u rke2-server -f
ssh ubuntu@host sudo journalctl -u rke2-agent -f
```

## Best Practices

1. **Use tags**: Control what runs with `-t` flag
2. **Version lock**: Specify exact versions for production
3. **Sensitive data**: Use Ansible vault for tokens/passwords
4. **Idempotent tasks**: All tasks should be safe to run multiple times
5. **Handlers**: Use handlers for service restarts (notified by changes)
6. **Testing**: Always run `--check` before production deployment

## Security Notes

- **rke2_token**: Change from default in `group_vars/rke2_cluster.yml`
- **SSH keys**: Use passphrase-protected keys in production
- **Vault**: Encrypt sensitive variables with `ansible-vault`
- **Inventory**: Don't commit credentials to version control

## References

- [Ansible Playbooks](https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_intro.html)
- [Ansible Roles](https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_reuse_roles.html)
- [RKE2 Documentation](https://docs.rke2.io/)
- [Rancher Documentation](https://rancher.com/docs/)
