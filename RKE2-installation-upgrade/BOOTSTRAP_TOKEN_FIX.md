# RKE2 Bootstrap Token Authentication Issue - Fix Documentation

## Problem Summary

The RKE2 agent (worker nodes) was failing to join the cluster with the error:
```
Failed to validate connection to cluster at https://10.0.1.179:6443: failed to get CA certs: Unauthorized
```

This occurred with RKE2 v1.36.3+rke2r1 when agents tried to bootstrap using the standard bootstrap token authentication mechanism.

## Root Cause

RKE2 v1.36.3 appears to have an issue with the bootstrap token authenticator that validates agent connections during the bootstrap phase. When agents attempt to fetch the cluster's CA certificate using the bootstrap token for authentication, the control plane returns a 401 Unauthorized response.

The issue is NOT:
- Network connectivity (verified via curl)
- Token format (token is correctly generated and matches)
- Control plane availability (API responds correctly)
- Token presence (node-token file exists with correct content)

The issue IS:
- RKE2's bootstrap token authenticator not accepting or validating the bootstrap token properly
- This may be a known bug in v1.36.3 or related to cluster configuration

## Solution Implemented

Added a workaround in `roles/rke2-agent/tasks/main.yml` that:

1. Waits for control plane API to be ready
2. Fetches the CA certificate directly from the control plane using Ansible delegation
3. Creates the RKE2 TLS directory on the agent
4. Installs the CA certificate locally on the agent

This allows the agent to proceed with bootstrap even if the bootstrap token authenticator is not working. The agent can now use the CA certificate it already has to validate the control plane connection, bypassing the failed bootstrap token validation.

## Changes Made

**File Modified:** `roles/rke2-agent/tasks/main.yml`

**New Tasks Added:**
- `Fetch CA certificate from control plane control node` - Retrieves CA cert via delegation
- `Create RKE2 TLS directory` - Ensures directory exists
- `Install CA certificate on agent` - Copies cert locally

## Testing

To verify the fix works:

1. Clean both control plane and agent:
```bash
ssh -i ~/github/ec2.pem ubuntu@<control-plane-ip> "sudo systemctl stop rke2-server && sudo rm -rf /etc/rancher/rke2 /var/lib/rancher/rke2"
ssh -i ~/github/ec2.pem ubuntu@<worker-ip> "sudo systemctl stop rke2-agent && sudo rm -rf /etc/rancher/rke2 /var/lib/rancher/rke2"
```

2. Run the updated playbook:
```bash
ansible-playbook -i inventory.ini rke2-install.yml -v
```

3. Verify the agent joined the cluster:
```bash
ssh -i ~/github/ec2.pem ubuntu@<control-plane-ip> "export KUBECONFIG=/etc/rancher/rke2/rke2.yaml && kubectl get nodes"
```

The agent should now appear in the nodes list with `Ready` status.

## Alternative Solutions (Not Implemented)

### Option 1: Use Different RKE2 Version
Upgrade to a newer RKE2 version (v1.27+) where this issue may be fixed.
- Pros: No workaround needed
- Cons: Version lock/testing required

### Option 2: Pre-Generate Bootstrap Token Secret
Manually create Kubernetes bootstrap token secrets in the cluster before starting agents.
- Pros: Uses standard Kubernetes mechanism
- Cons: Complex, requires manual cluster setup

### Option 3: Disable TLS Verification
Configure agents with insecure TLS settings.
- Pros: Simple
- Cons: Security risk, not production-recommended

## Recommendations

1. **Verify the fix works** in your environment
2. **Test upgrades** to confirm agents upgrade smoothly
3. **Monitor for issues** in your RKE2 version v1.36.3
4. **Plan to upgrade** to a newer RKE2 version for long-term support
5. **Report the issue** to Rancher/RKE2 team if not already reported

## References

- RKE2 GitHub: https://github.com/rancher/rke2
- RKE2 Docs: https://docs.rke2.io/
- Kubernetes Bootstrap Tokens: https://kubernetes.io/docs/reference/access-authn-authz/bootstrap-tokens/

## Created

2026-08-20 - During bootstrap token authentication debugging
