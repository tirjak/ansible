# SSH Host Key Management - Auto-Accept Solution

## Problem Solved
No more manual `ssh-keyscan` commands needed for new EC2 instances!

## How It Works

### 1. **Pre-Task in Playbook**
The `rke2-install.yml` now includes a pre-task that:
- Automatically scans host keys from each instance
- Adds them to `~/.ssh/known_hosts` 
- Runs before any SSH connection attempts
- Prevents "Host key verification failed" errors

```yaml
pre_tasks:
  - name: Add host keys to known_hosts automatically
    ansible.builtin.known_hosts:
      name: "{{ ansible_host }}"
      key: "{{ lookup('pipe', 'ssh-keyscan -H ' + ansible_host + ' 2>/dev/null') }}"
      state: present
    become: no
    delegate_to: localhost
```

### 2. **SSH Configuration (ansible.cfg)**
Key setting: `StrictHostKeyChecking=accept-new`

This tells SSH to:
- Accept new host keys automatically
- Not prompt for verification
- Still verify subsequent connections against known_hosts
- Provides security while avoiding manual steps

## What Changed

### Files Modified:
- `rke2-install.yml` - Added pre-task for host key scanning
- `ansible.cfg` - Created with SSH and Ansible best practices

### Configuration Highlights:
```ini
[defaults]
host_key_checking = True
ssh_args = -C -o ControlMaster=auto -o ControlPersist=60s -o StrictHostKeyChecking=accept-new
timeout = 30
connect_timeout = 10
```

## Usage

Now you can simply run:

```bash
cd /Users/tirjakmohapatra/github/ansible/RKE2-installation-upgrade
ansible-playbook -i inventory.ini rke2-install.yml -v
```

**No need to manually add host keys ever again!**

The playbook will:
1. ✅ Auto-scan and add host keys from inventory
2. ✅ Connect via SSH
3. ✅ Run the full installation

## Benefits

| Aspect | Before | After |
|--------|--------|-------|
| Manual steps | Required ssh-keyscan for each new instance | Zero manual steps |
| Time | Slow, error-prone | Fast, automated |
| Scalability | Doesn't scale to many instances | Works with any number of instances |
| Security | Still verified | Still verified via known_hosts |

## How It Stays Secure

1. **First connection**: SSH accepts new key via `StrictHostKeyChecking=accept-new`
2. **Key is saved**: Added to `~/.ssh/known_hosts`
3. **Subsequent connections**: SSH verifies key matches known_hosts
4. **Man-in-the-middle protection**: Maintained after initial connection

## Troubleshooting

If you still get "Host key verification failed":

1. **Check ansible.cfg exists** in the project directory:
   ```bash
   ls -la ansible.cfg
   ```

2. **Check SSH connectivity manually**:
   ```bash
   ssh -i ~/github/ec2.pem ubuntu@<instance-ip> "echo connected"
   ```

3. **Verify inventory.ini** has correct IPs and SSH key path:
   ```bash
   cat inventory.ini
   ```

## For Future Projects

Copy these settings to any new Ansible project:
- Use the same `ansible.cfg` format
- Add the pre-task to any playbook using dynamic inventory IPs
- Works with any infrastructure (AWS, Azure, GCP, etc.)
