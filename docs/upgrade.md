# RKE2 Cluster Upgrade Guide

This guide covers upgrading RKE2 clusters deployed with this Ansible collection.

## Table of Contents

- [Overview](#overview)
- [Pre-Upgrade Checklist](#pre-upgrade-checklist)
- [Upgrade Methods](#upgrade-methods)
- [Rolling Upgrade Procedure](#rolling-upgrade-procedure)
- [Rollback Procedure](#rollback-procedure)
- [Version Compatibility](#version-compatibility)

## Overview

RKE2 upgrades should be performed carefully to minimize cluster downtime. The general strategy is:

1. **Backup** the cluster before any upgrade
2. **Upgrade servers** one at a time (control plane)
3. **Upgrade agents** (worker nodes)
4. **Verify** cluster health after each node

## Pre-Upgrade Checklist

Before starting the upgrade:

- [ ] Review [RKE2 release notes](https://github.com/rancher/rke2/releases) for breaking changes
- [ ] Create etcd snapshot backup
- [ ] Verify current cluster health
- [ ] Plan maintenance window
- [ ] Test upgrade in staging environment first
- [ ] Ensure sufficient resources for rolling upgrade

### Create Pre-Upgrade Backup

```bash
# On any server node
/var/lib/rancher/rke2/bin/rke2 etcd-snapshot save \
  --name pre-upgrade-$(date +%Y%m%d-%H%M%S)
```

### Verify Cluster Health

```bash
# Check all nodes are Ready
kubectl get nodes

# Check system pods
kubectl get pods -n kube-system

# Check etcd health (on server nodes)
/var/lib/rancher/rke2/bin/etcdctl \
  --cacert=/var/lib/rancher/rke2/server/tls/etcd/server-ca.crt \
  --cert=/var/lib/rancher/rke2/server/tls/etcd/server-client.crt \
  --key=/var/lib/rancher/rke2/server/tls/etcd/server-client.key \
  endpoint health --cluster
```

## Upgrade Methods

### Method 1: Using Ansible (Recommended)

Update the version in your inventory:

```yaml
# inventory/group_vars/all.yml
rke2_install_version: "v1.30.0+rke2r1"  # New version
```

Run the playbook with serial execution:

```bash
# Upgrade servers first (one at a time)
ansible-playbook site.yml -i inventory/hosts.yml -b \
  --limit rke2_servers \
  -e "serial=1"

# Then upgrade agents
ansible-playbook site.yml -i inventory/hosts.yml -b \
  --limit rke2_agents \
  -e "serial=1"
```

### Method 2: Manual Upgrade

For more control, upgrade nodes manually:

```bash
# On each node, stop RKE2
systemctl stop rke2-server  # or rke2-agent

# Install new version
curl -sfL https://get.rke2.io | INSTALL_RKE2_VERSION=v1.30.0+rke2r1 sh -

# Start service
systemctl start rke2-server  # or rke2-agent

# Verify node is Ready
kubectl get nodes
```

### Method 3: Channel-Based Upgrade

Use RKE2 channels for automatic version selection:

```yaml
# inventory/group_vars/all.yml
rke2_channel: "stable"  # or "latest", "v1.30"
rke2_install_version: ""  # Empty to use channel
```

## Rolling Upgrade Procedure

### Step 1: Upgrade First Server

```bash
# Drain the node (optional for HA clusters)
kubectl drain server1 --ignore-daemonsets --delete-emptydir-data

# Upgrade using Ansible
ansible-playbook site.yml -i inventory/hosts.yml -b --limit server1

# Uncordon the node
kubectl uncordon server1

# Verify node is Ready and version updated
kubectl get nodes -o wide
```

### Step 2: Upgrade Remaining Servers

Repeat for each server node:

```bash
# Wait for previous node to be fully Ready
kubectl wait --for=condition=Ready node/server1 --timeout=300s

# Proceed with next server
ansible-playbook site.yml -i inventory/hosts.yml -b --limit server2
```

### Step 3: Upgrade Agent Nodes

```bash
# Upgrade agents (can be done in parallel for large clusters)
ansible-playbook site.yml -i inventory/hosts.yml -b \
  --limit rke2_agents \
  -e "serial=3"  # 3 nodes at a time
```

### Step 4: Post-Upgrade Verification

```bash
# Verify all nodes upgraded
kubectl get nodes -o wide

# Check system pods
kubectl get pods -n kube-system

# Run smoke tests
kubectl run test --image=nginx --restart=Never
kubectl delete pod test
```

## Rollback Procedure

If upgrade fails, rollback to previous version:

### Option 1: Restore from Snapshot

```bash
# Stop RKE2 on all servers
ansible -i inventory/hosts.yml rke2_servers -a "systemctl stop rke2-server" -b

# On first server, restore snapshot
/var/lib/rancher/rke2/bin/rke2 server \
  --cluster-reset \
  --cluster-reset-restore-path=/var/lib/rancher/rke2/server/db/snapshots/pre-upgrade-xxx

# Start first server
systemctl start rke2-server

# Rejoin other servers (they will sync from restored state)
ansible-playbook site.yml -i inventory/hosts.yml -b --limit 'rke2_servers:!server1'
```

### Option 2: Reinstall Previous Version

```bash
# Update inventory with previous version
# inventory/group_vars/all.yml
rke2_install_version: "v1.29.0+rke2r1"  # Previous version

# Run playbook
ansible-playbook site.yml -i inventory/hosts.yml -b
```

## Version Compatibility

### Supported Upgrade Paths

| From | To | Supported |
|------|-----|-----------|
| v1.28.x | v1.29.x | Yes |
| v1.29.x | v1.30.x | Yes |
| v1.28.x | v1.30.x | Yes (skip allowed) |

### Kubernetes Version Skew Policy

- **kube-apiserver**: Can be 1 minor version ahead of kubelet
- **kubelet**: Cannot be newer than kube-apiserver
- **Recommended**: Upgrade all nodes within same maintenance window

### Component Compatibility

Always verify compatibility with:
- CNI plugins (Canal, Calico, Cilium)
- Ingress controllers
- Storage drivers
- Monitoring stack

## Upgrade Automation Script

Create a script for automated rolling upgrades:

```bash
#!/bin/bash
# upgrade-cluster.sh

set -e

NEW_VERSION="${1:-v1.30.0+rke2r1}"
INVENTORY="inventory/hosts.yml"

echo "=== RKE2 Cluster Upgrade to $NEW_VERSION ==="

# Pre-upgrade backup
echo "Creating pre-upgrade snapshot..."
ansible -i $INVENTORY rke2_servers[0] -a \
  "/var/lib/rancher/rke2/bin/rke2 etcd-snapshot save --name pre-upgrade-$(date +%Y%m%d-%H%M%S)" -b

# Update version in inventory
echo "Updating version in inventory..."
sed -i "s/rke2_install_version:.*/rke2_install_version: \"$NEW_VERSION\"/" inventory/group_vars/all.yml

# Upgrade servers
echo "Upgrading server nodes..."
for server in $(ansible-inventory -i $INVENTORY --list | jq -r '.rke2_servers.hosts[]'); do
  echo "Upgrading $server..."
  ansible-playbook site.yml -i $INVENTORY -b --limit $server

  echo "Waiting for $server to be Ready..."
  kubectl wait --for=condition=Ready node/$server --timeout=300s
done

# Upgrade agents
echo "Upgrading agent nodes..."
ansible-playbook site.yml -i $INVENTORY -b --limit rke2_agents

# Verify
echo "Verifying cluster health..."
kubectl get nodes -o wide
kubectl get pods -n kube-system

echo "=== Upgrade Complete ==="
```

## Troubleshooting Upgrades

### Node Stuck in NotReady

```bash
# Check kubelet logs
journalctl -u rke2-server -f  # or rke2-agent

# Check for pending pods
kubectl get pods -A | grep -v Running
```

### etcd Issues After Upgrade

```bash
# Check etcd logs
journalctl -u rke2-server | grep etcd

# Verify etcd members
/var/lib/rancher/rke2/bin/etcdctl member list
```

### Rollback Single Node

```bash
# Stop RKE2
systemctl stop rke2-server

# Remove current installation
/var/lib/rancher/rke2/bin/rke2-uninstall.sh

# Install previous version
curl -sfL https://get.rke2.io | INSTALL_RKE2_VERSION=v1.29.0+rke2r1 sh -

# Start service
systemctl start rke2-server
```
