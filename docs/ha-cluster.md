# High Availability Cluster Configuration

This guide covers deploying highly available RKE2 clusters with multiple control plane nodes.

## Table of Contents

- [Overview](#overview)
- [Requirements](#requirements)
- [Basic HA Setup (3 Servers)](#basic-ha-setup-3-servers)
- [HA with Dedicated etcd](#ha-with-dedicated-etcd)
- [Multi-Zone Deployment](#multi-zone-deployment)
- [Load Balancer Configuration](#load-balancer-configuration)
- [Best Practices](#best-practices)

## Overview

A highly available RKE2 cluster requires:
- **Minimum 3 server nodes** for etcd quorum
- **Odd number of servers** (3, 5, 7) for proper leader election
- **Load balancer** in front of the API servers (recommended)

## Requirements

| Component | Minimum | Recommended |
|-----------|---------|-------------|
| Server nodes | 3 | 3 or 5 |
| CPU per server | 2 cores | 4 cores |
| RAM per server | 4 GB | 8 GB |
| Disk per server | 50 GB SSD | 100 GB SSD |

## Basic HA Setup (3 Servers)

### Inventory

```yaml
# inventory/hosts.yml
---
rke2_cluster:
  children:
    rke2_servers:
      hosts:
        server1.example.com:
          ansible_host: 10.0.0.10
        server2.example.com:
          ansible_host: 10.0.0.11
        server3.example.com:
          ansible_host: 10.0.0.12
    rke2_agents:
      hosts:
        worker1.example.com:
          ansible_host: 10.0.0.20
        worker2.example.com:
          ansible_host: 10.0.0.21
        worker3.example.com:
          ansible_host: 10.0.0.22
```

### Cluster Configuration

```yaml
# inventory/group_vars/all.yml
---
# Use first server as initial bootstrap
rke2_kubernetes_api_server_host: "10.0.0.10"

# Or use a load balancer VIP
# rke2_kubernetes_api_server_host: "10.0.0.100"

cluster_rke2_config:
  # Recommended for HA clusters
  cluster-cidr: "10.42.0.0/16"
  service-cidr: "10.43.0.0/16"
```

### Server-specific Configuration

```yaml
# inventory/group_vars/rke2_servers.yml
---
group_rke2_config:
  # Enable etcd snapshots for backup
  etcd-snapshot-schedule-cron: "0 */6 * * *"
  etcd-snapshot-retention: 5

  # TLS SANs for load balancer
  tls-san:
    - "10.0.0.100"  # Load balancer VIP
    - "k8s.example.com"  # DNS name
```

## HA with Dedicated etcd

For larger clusters, you can run etcd on dedicated nodes:

```yaml
# inventory/hosts.yml
---
rke2_cluster:
  children:
    rke2_servers:
      hosts:
        server1.example.com:
          ansible_host: 10.0.0.10
        server2.example.com:
          ansible_host: 10.0.0.11
        server3.example.com:
          ansible_host: 10.0.0.12
    rke2_agents:
      hosts:
        worker[1:10].example.com:
```

```yaml
# inventory/group_vars/rke2_servers.yml
---
group_rke2_config:
  # Disable workload scheduling on control plane
  node-taint:
    - "CriticalAddonsOnly=true:NoExecute"
```

## Multi-Zone Deployment

Spread nodes across availability zones for resilience:

```yaml
# inventory/hosts.yml
---
rke2_cluster:
  children:
    rke2_servers:
      hosts:
        server1-zone-a.example.com:
          ansible_host: 10.0.1.10
          zone: zone-a
        server2-zone-b.example.com:
          ansible_host: 10.0.2.10
          zone: zone-b
        server3-zone-c.example.com:
          ansible_host: 10.0.3.10
          zone: zone-c
    rke2_agents:
      children:
        zone_a_workers:
          hosts:
            worker1-zone-a.example.com:
              ansible_host: 10.0.1.20
        zone_b_workers:
          hosts:
            worker1-zone-b.example.com:
              ansible_host: 10.0.2.20
        zone_c_workers:
          hosts:
            worker1-zone-c.example.com:
              ansible_host: 10.0.3.20
```

```yaml
# inventory/host_vars/server1-zone-a.example.com.yml
---
host_rke2_config:
  node-label:
    - "topology.kubernetes.io/zone=zone-a"
```

## Load Balancer Configuration

### HAProxy Example

```
# /etc/haproxy/haproxy.cfg
frontend k8s-api
    bind *:6443
    mode tcp
    option tcplog
    default_backend k8s-api-backend

backend k8s-api-backend
    mode tcp
    option tcp-check
    balance roundrobin
    server server1 10.0.0.10:6443 check fall 3 rise 2
    server server2 10.0.0.11:6443 check fall 3 rise 2
    server server3 10.0.0.12:6443 check fall 3 rise 2

frontend rke2-supervisor
    bind *:9345
    mode tcp
    option tcplog
    default_backend rke2-supervisor-backend

backend rke2-supervisor-backend
    mode tcp
    option tcp-check
    balance roundrobin
    server server1 10.0.0.10:9345 check fall 3 rise 2
    server server2 10.0.0.11:9345 check fall 3 rise 2
    server server3 10.0.0.12:9345 check fall 3 rise 2
```

### Nginx Example

```nginx
# /etc/nginx/nginx.conf
stream {
    upstream k8s_api {
        least_conn;
        server 10.0.0.10:6443 max_fails=3 fail_timeout=30s;
        server 10.0.0.11:6443 max_fails=3 fail_timeout=30s;
        server 10.0.0.12:6443 max_fails=3 fail_timeout=30s;
    }

    upstream rke2_supervisor {
        least_conn;
        server 10.0.0.10:9345 max_fails=3 fail_timeout=30s;
        server 10.0.0.11:9345 max_fails=3 fail_timeout=30s;
        server 10.0.0.12:9345 max_fails=3 fail_timeout=30s;
    }

    server {
        listen 6443;
        proxy_pass k8s_api;
    }

    server {
        listen 9345;
        proxy_pass rke2_supervisor;
    }
}
```

### Using Load Balancer with Ansible

```yaml
# inventory/group_vars/all.yml
---
# Point to load balancer instead of first server
rke2_kubernetes_api_server_host: "lb.example.com"
```

```yaml
# inventory/group_vars/rke2_servers.yml
---
group_rke2_config:
  tls-san:
    - "lb.example.com"
    - "10.0.0.100"  # LB IP
```

## Best Practices

### 1. Use Odd Number of Servers

Always use 3, 5, or 7 server nodes:
- **3 nodes**: Tolerates 1 failure
- **5 nodes**: Tolerates 2 failures
- **7 nodes**: Tolerates 3 failures

### 2. Spread Across Failure Domains

Distribute server nodes across:
- Different racks
- Different availability zones
- Different data centers (with low latency)

### 3. Configure etcd Snapshots

```yaml
group_rke2_config:
  etcd-snapshot-schedule-cron: "0 */4 * * *"  # Every 4 hours
  etcd-snapshot-retention: 10
  etcd-snapshot-dir: "/var/lib/rancher/rke2/server/db/snapshots"
```

### 4. Set Resource Reservations

```yaml
group_rke2_config:
  kubelet-arg:
    - "system-reserved=cpu=500m,memory=500Mi"
    - "kube-reserved=cpu=500m,memory=500Mi"
```

### 5. Enable Pod Security

```yaml
cluster_rke2_config:
  profile: cis  # Or cis-1.23, cis-1.6
```

### 6. Use Dedicated Control Plane

For production, taint control plane nodes:

```yaml
# inventory/group_vars/rke2_servers.yml
group_rke2_config:
  node-taint:
    - "node-role.kubernetes.io/control-plane=true:NoSchedule"
```

### 7. Monitor etcd Health

Add monitoring for etcd:

```bash
# Check etcd member list
/var/lib/rancher/rke2/bin/etcdctl \
  --cacert=/var/lib/rancher/rke2/server/tls/etcd/server-ca.crt \
  --cert=/var/lib/rancher/rke2/server/tls/etcd/server-client.crt \
  --key=/var/lib/rancher/rke2/server/tls/etcd/server-client.key \
  member list

# Check etcd endpoint health
/var/lib/rancher/rke2/bin/etcdctl \
  --cacert=/var/lib/rancher/rke2/server/tls/etcd/server-ca.crt \
  --cert=/var/lib/rancher/rke2/server/tls/etcd/server-client.crt \
  --key=/var/lib/rancher/rke2/server/tls/etcd/server-client.key \
  endpoint health --cluster
```
