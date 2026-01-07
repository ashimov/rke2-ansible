# Troubleshooting Guide

This guide covers common issues and their solutions when deploying RKE2 clusters with this Ansible collection.

## Table of Contents

- [Node Join Failures](#node-join-failures)
- [API Server Issues](#api-server-issues)
- [Token Problems](#token-problems)
- [CIS Hardening Issues](#cis-hardening-issues)
- [Network Issues](#network-issues)
- [Certificate Problems](#certificate-problems)
- [Collecting Diagnostic Information](#collecting-diagnostic-information)

## Node Join Failures

### Symptom: Node fails to join the cluster

**Possible Causes:**

1. **Token mismatch or expiration**
   ```bash
   # Check token on server node
   cat /var/lib/rancher/rke2/server/node-token

   # Verify token in agent config
   cat /etc/rancher/rke2/config.yaml
   ```

2. **Network connectivity issues**
   ```bash
   # Test connectivity to API server from agent
   curl -k https://<server-ip>:9345/cacerts

   # Check firewall rules
   iptables -L -n | grep -E '(6443|9345)'
   ```

3. **DNS resolution problems**
   ```bash
   # Verify hostname resolution
   getent hosts <server-hostname>
   ```

**Solution:**

Ensure the `rke2_kubernetes_api_server_host` variable points to an accessible server:

```yaml
# inventory/group_vars/all.yml
rke2_kubernetes_api_server_host: "10.0.0.10"  # Use IP if DNS is unreliable
```

### Symptom: Timeout waiting for node to become Ready

**Possible Causes:**

1. Slow network or disk I/O
2. Insufficient resources (CPU/RAM)
3. Container images still downloading

**Solution:**

Increase timeout values:

```yaml
# inventory/group_vars/all.yml
rke2_api_wait_timeout: 600
rke2_node_ready_timeout: 600
rke2_node_ready_retries: 60
rke2_node_ready_delay: 15
```

## API Server Issues

### Symptom: API server not responding

**Check service status:**

```bash
systemctl status rke2-server
journalctl -u rke2-server -f
```

**Check if port is listening:**

```bash
ss -tlnp | grep 6443
```

**Common fixes:**

1. Restart the service:
   ```bash
   systemctl restart rke2-server
   ```

2. Check etcd health:
   ```bash
   /var/lib/rancher/rke2/bin/etcdctl \
     --cacert=/var/lib/rancher/rke2/server/tls/etcd/server-ca.crt \
     --cert=/var/lib/rancher/rke2/server/tls/etcd/server-client.crt \
     --key=/var/lib/rancher/rke2/server/tls/etcd/server-client.key \
     endpoint health
   ```

## Token Problems

### Symptom: Token file not found

The token file is created after the first server starts. If it doesn't exist:

```bash
# Check if RKE2 server is running
systemctl status rke2-server

# Token should be at
ls -la /var/lib/rancher/rke2/server/node-token
```

### Symptom: Token retrieval timeout

Increase the API wait timeout:

```yaml
rke2_api_wait_timeout: 600
```

## CIS Hardening Issues

### Symptom: Playbook fails during CIS hardening

**Check if sysctl config exists:**

```bash
# For RPM install
ls -la /usr/share/rke2/rke2-cis-sysctl.conf

# For tarball install
ls -la /usr/local/share/rke2/rke2-cis-sysctl.conf
```

### Symptom: Node requires reboot after CIS changes

This is expected behavior. The playbook will automatically reboot the node if:
- CIS profile is enabled
- sysctl configuration changed
- RKE2 was already running

To adjust reboot timeout:

```yaml
rke2_reboot_timeout: 600
```

## Network Issues

### Symptom: Pods cannot communicate across nodes

**Check CNI configuration:**

```bash
# Verify CNI plugins
ls -la /var/lib/rancher/rke2/agent/etc/cni/net.d/

# Check Flannel or Canal status
kubectl get pods -n kube-system | grep -E '(flannel|canal)'
```

**Check iptables rules:**

```bash
iptables -L -n -t nat
iptables -L -n -t filter
```

### Symptom: Service discovery not working

```bash
# Check CoreDNS status
kubectl get pods -n kube-system -l k8s-app=kube-dns

# Test DNS resolution
kubectl run -it --rm debug --image=busybox -- nslookup kubernetes.default
```

## Certificate Problems

### Symptom: Certificate expired or invalid

**Check certificate expiration:**

```bash
openssl x509 -in /var/lib/rancher/rke2/server/tls/server-ca.crt -noout -dates
```

**Rotate certificates:**

```bash
# Stop RKE2
systemctl stop rke2-server

# Remove old certificates
rm -rf /var/lib/rancher/rke2/server/tls

# Start RKE2 (will regenerate certificates)
systemctl start rke2-server
```

## Collecting Diagnostic Information

### Basic information gathering

```bash
# System information
uname -a
cat /etc/os-release

# RKE2 version
/var/lib/rancher/rke2/bin/rke2 --version

# Service status
systemctl status rke2-server rke2-agent

# Recent logs
journalctl -u rke2-server --since "1 hour ago"
journalctl -u rke2-agent --since "1 hour ago"
```

### Kubernetes cluster state

```bash
# Node status
kubectl get nodes -o wide

# All pods status
kubectl get pods -A

# Events
kubectl get events -A --sort-by='.lastTimestamp'
```

### Network diagnostics

```bash
# Check listening ports
ss -tlnp

# Check iptables
iptables-save > /tmp/iptables-rules.txt

# Check routes
ip route show
```

### Create diagnostic bundle

```bash
#!/bin/bash
BUNDLE_DIR="/tmp/rke2-diagnostics-$(date +%Y%m%d-%H%M%S)"
mkdir -p "$BUNDLE_DIR"

# System info
uname -a > "$BUNDLE_DIR/system-info.txt"
cat /etc/os-release >> "$BUNDLE_DIR/system-info.txt"

# RKE2 config
cp /etc/rancher/rke2/config.yaml "$BUNDLE_DIR/" 2>/dev/null

# Service logs
journalctl -u rke2-server --since "2 hours ago" > "$BUNDLE_DIR/rke2-server.log" 2>/dev/null
journalctl -u rke2-agent --since "2 hours ago" > "$BUNDLE_DIR/rke2-agent.log" 2>/dev/null

# Kubernetes state
kubectl get nodes -o wide > "$BUNDLE_DIR/nodes.txt" 2>/dev/null
kubectl get pods -A > "$BUNDLE_DIR/pods.txt" 2>/dev/null
kubectl get events -A --sort-by='.lastTimestamp' > "$BUNDLE_DIR/events.txt" 2>/dev/null

# Create archive
tar -czf "$BUNDLE_DIR.tar.gz" -C /tmp "$(basename $BUNDLE_DIR)"
echo "Diagnostic bundle created: $BUNDLE_DIR.tar.gz"
```

## fapolicyd on RHEL 8+

If you're using fapolicyd, add these rules before installation:

```bash
cat <<-EOF >> /etc/fapolicyd/rules.d/80-rke2.rules
allow perm=any all : dir=/var/lib/rancher/
allow perm=any all : dir=/opt/cni/
allow perm=any all : dir=/run/k3s/
allow perm=any all : dir=/var/lib/kubelet/
EOF

systemctl restart fapolicyd
```

## Getting Help

If you're still experiencing issues:

1. Check the [RKE2 documentation](https://docs.rke2.io/)
2. Search [GitHub Issues](https://github.com/ashimov/rke2-ansible/issues)
3. Open a new issue with your diagnostic bundle
