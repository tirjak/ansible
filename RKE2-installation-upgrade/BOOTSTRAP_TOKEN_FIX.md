# RKE2 Agent Join Failure - Root Cause and Fix

## Problem

Worker (agent) nodes failed to join the cluster with:
```
Failed to validate connection to cluster at https://<control-plane-ip>:6443: failed to get CA certs: Unauthorized
```

## Root Cause

RKE2 exposes two separate HTTPS listeners on the control plane:

- **Port 6443** - the Kubernetes API server. Requires an authenticated kubeconfig; rejects unauthenticated requests (including `/cacerts`) with `401 Unauthorized`.
- **Port 9345** - the RKE2 **supervisor** port. This is what agents use to bootstrap: fetch the CA certificate, register, and receive cluster config. It answers `/cacerts` without authentication (confirmed via `curl -sk https://<ip>:9345/cacerts` returning `200` with the CA cert).

`roles/rke2-agent/defaults/main.yml` had agents pointed at port 6443:

```yaml
rke2_server_url: "https://{{ rke2_control_plane_ip }}:6443"
```

Agents therefore hit the Kubernetes API server instead of the supervisor, which correctly returned 401 for the unauthenticated `/cacerts` call. This produced the exact error seen, and had nothing to do with the token value, the CA certificate content, or network connectivity - all of which were verified correct during debugging.

## Fix

Changed the agent server URL to use port 9345:

```yaml
# roles/rke2-agent/defaults/main.yml
rke2_server_url: "https://{{ rke2_control_plane_ip }}:9345"
```

Verified directly on a worker node before applying the fix:

```bash
curl -sk -o /dev/null -w '%{http_code}\n' https://<control-plane-ip>:6443/cacerts   # 401
curl -sk -o /dev/null -w '%{http_code}\n' https://<control-plane-ip>:9345/cacerts   # 200
```

## Infrastructure note

Your EC2 security group / firewall must allow port **9345** between worker nodes and the control plane, in addition to 6443 and 10250. Add this to your security group rules if it isn't already open.

## Dead ends ruled out during debugging (for reference)

- Token mismatch between control plane and agent config - confirmed identical, not the cause.
- CA certificate not present on the agent - manually pre-installing the CA cert on the agent did not fix the issue, confirming the failure was in the network call to the wrong port, not in local cert trust.
- Stale cluster state - reproduced on a fully clean install (control plane and worker wiped with the official `rke2-uninstall.sh`), ruling out leftover state from earlier attempts.
