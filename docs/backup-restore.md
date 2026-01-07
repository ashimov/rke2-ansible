# Backup and Restore Guide

This guide covers backup and restore procedures for RKE2 clusters deployed with this Ansible collection.

## Table of Contents

- [Overview](#overview)
- [What to Backup](#what-to-backup)
- [Automated etcd Snapshots](#automated-etcd-snapshots)
- [Manual Backup](#manual-backup)
- [Restore Procedures](#restore-procedures)
- [Disaster Recovery](#disaster-recovery)

## Overview

RKE2 stores all cluster state in etcd. Regular backups are essential for disaster recovery. RKE2 provides built-in etcd snapshot functionality.

## What to Backup

| Component | Location | Priority |
|-----------|----------|----------|
| etcd data | `/var/lib/rancher/rke2/server/db/` | Critical |
| etcd snapshots | `/var/lib/rancher/rke2/server/db/snapshots/` | Critical |
| RKE2 config | `/etc/rancher/rke2/config.yaml` | High |
| TLS certificates | `/var/lib/rancher/rke2/server/tls/` | High |
| Token | `/var/lib/rancher/rke2/server/node-token` | High |
| Custom manifests | `/var/lib/rancher/rke2/server/manifests/` | Medium |

## Automated etcd Snapshots

### Enable Automatic Snapshots

Configure automatic snapshots in your inventory:

```yaml
# inventory/group_vars/rke2_servers.yml
---
group_rke2_config:
  # Take snapshot every 6 hours
  etcd-snapshot-schedule-cron: "0 */6 * * *"

  # Keep last 10 snapshots
  etcd-snapshot-retention: 10

  # Custom snapshot directory (optional)
  etcd-snapshot-dir: "/var/lib/rancher/rke2/server/db/snapshots"

  # Snapshot name prefix (optional)
  etcd-snapshot-name: "rke2-snapshot"
```

### Verify Automatic Snapshots

```bash
# List snapshots
ls -la /var/lib/rancher/rke2/server/db/snapshots/

# Or using rke2 command
/var/lib/rancher/rke2/bin/rke2 etcd-snapshot list
```

## Manual Backup

### Create On-Demand Snapshot

```bash
# On any server node
/var/lib/rancher/rke2/bin/rke2 etcd-snapshot save --name pre-upgrade-$(date +%Y%m%d-%H%M%S)
```

### Backup Script

Create a comprehensive backup script:

```bash
#!/bin/bash
# /usr/local/bin/rke2-backup.sh

set -e

BACKUP_DIR="/backup/rke2/$(date +%Y%m%d-%H%M%S)"
SNAPSHOT_NAME="manual-backup-$(date +%Y%m%d-%H%M%S)"

# Create backup directory
mkdir -p "$BACKUP_DIR"

echo "Creating etcd snapshot..."
/var/lib/rancher/rke2/bin/rke2 etcd-snapshot save --name "$SNAPSHOT_NAME"

echo "Copying snapshot to backup location..."
cp /var/lib/rancher/rke2/server/db/snapshots/${SNAPSHOT_NAME}* "$BACKUP_DIR/"

echo "Backing up configuration..."
cp /etc/rancher/rke2/config.yaml "$BACKUP_DIR/" 2>/dev/null || true

echo "Backing up token..."
cp /var/lib/rancher/rke2/server/node-token "$BACKUP_DIR/" 2>/dev/null || true

echo "Backing up TLS certificates..."
tar -czf "$BACKUP_DIR/tls-certs.tar.gz" -C /var/lib/rancher/rke2/server tls/ 2>/dev/null || true

echo "Creating archive..."
tar -czf "${BACKUP_DIR}.tar.gz" -C "$(dirname $BACKUP_DIR)" "$(basename $BACKUP_DIR)"
rm -rf "$BACKUP_DIR"

echo "Backup complete: ${BACKUP_DIR}.tar.gz"
```

### Offsite Backup

Copy backups to remote storage:

```bash
# Using rsync
rsync -avz /backup/rke2/ backup-server:/rke2-backups/

# Using AWS S3
aws s3 sync /var/lib/rancher/rke2/server/db/snapshots/ s3://my-bucket/rke2-snapshots/

# Using rclone
rclone sync /var/lib/rancher/rke2/server/db/snapshots/ remote:rke2-backups/
```

## Restore Procedures

### Restore from Snapshot (Same Cluster)

Use this when etcd is corrupted but the server is intact:

```bash
# Stop RKE2 on all servers
systemctl stop rke2-server

# On the first server, restore from snapshot
/var/lib/rancher/rke2/bin/rke2 server \
  --cluster-reset \
  --cluster-reset-restore-path=/var/lib/rancher/rke2/server/db/snapshots/<snapshot-name>

# Start RKE2
systemctl start rke2-server

# Verify cluster
kubectl get nodes
```

### Restore to New Cluster

Use this for disaster recovery to new infrastructure:

1. **Prepare new server**

```bash
# Install RKE2 (do not start yet)
curl -sfL https://get.rke2.io | sh -
```

2. **Copy snapshot and config**

```bash
# Create directories
mkdir -p /var/lib/rancher/rke2/server/db/snapshots
mkdir -p /etc/rancher/rke2

# Copy snapshot
scp backup-server:/backups/snapshot-xxx.db /var/lib/rancher/rke2/server/db/snapshots/

# Copy config (modify as needed for new IPs)
scp backup-server:/backups/config.yaml /etc/rancher/rke2/
```

3. **Restore cluster**

```bash
# Start with restore
/var/lib/rancher/rke2/bin/rke2 server \
  --cluster-reset \
  --cluster-reset-restore-path=/var/lib/rancher/rke2/server/db/snapshots/<snapshot-name>
```

4. **Verify and rejoin nodes**

```bash
# Check cluster
kubectl get nodes

# Rejoin other servers and agents using Ansible
ansible-playbook site.yml -i inventory/hosts.yml -b
```

### Restore Single etcd Member

For HA clusters when one etcd member fails:

```bash
# On the failed server, stop RKE2
systemctl stop rke2-server

# Remove etcd data
rm -rf /var/lib/rancher/rke2/server/db/etcd

# Start RKE2 (will rejoin cluster)
systemctl start rke2-server

# Verify etcd membership
/var/lib/rancher/rke2/bin/etcdctl \
  --cacert=/var/lib/rancher/rke2/server/tls/etcd/server-ca.crt \
  --cert=/var/lib/rancher/rke2/server/tls/etcd/server-client.crt \
  --key=/var/lib/rancher/rke2/server/tls/etcd/server-client.key \
  member list
```

## Disaster Recovery

### Complete Cluster Loss

If all servers are lost:

1. **Provision new infrastructure**
2. **Copy most recent snapshot to first server**
3. **Restore using single-server restore procedure**
4. **Rejoin additional servers**
5. **Verify all workloads**

### Recovery Checklist

- [ ] Latest backup available
- [ ] Backup integrity verified
- [ ] New infrastructure provisioned
- [ ] Network connectivity confirmed
- [ ] DNS records updated (if needed)
- [ ] Snapshot restored
- [ ] All nodes rejoined
- [ ] System pods healthy
- [ ] Workloads running
- [ ] Ingress/LoadBalancer working
- [ ] Persistent volumes accessible

### Testing Backups

Regularly test your backup procedures:

```bash
# Create test snapshot
/var/lib/rancher/rke2/bin/rke2 etcd-snapshot save --name test-backup

# Verify snapshot integrity (on a test system)
/var/lib/rancher/rke2/bin/rke2 server \
  --cluster-reset \
  --cluster-reset-restore-path=/path/to/test-backup
```

## Best Practices

1. **Backup frequency**: At least every 6 hours for production
2. **Retention**: Keep at least 7 days of snapshots
3. **Offsite storage**: Always copy backups to remote location
4. **Test restores**: Monthly restore tests to verify backup integrity
5. **Document procedures**: Keep runbooks for common scenarios
6. **Monitor backups**: Alert on backup failures
7. **Encrypt backups**: Protect sensitive data in transit and at rest

## Monitoring Backup Status

Add monitoring for backup health:

```yaml
# Prometheus alert example
groups:
  - name: rke2-backup
    rules:
      - alert: RKE2BackupMissing
        expr: time() - rke2_etcd_snapshot_last_success_timestamp > 86400
        for: 1h
        labels:
          severity: warning
        annotations:
          summary: "RKE2 etcd backup not taken in 24 hours"
```
