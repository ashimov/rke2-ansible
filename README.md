<h1 align="center">RKE2 Ansible Collection</h1>

<p align="center">
  <strong>Production-ready Ansible automation for RKE2 Kubernetes clusters</strong>
</p>

<p align="center">
  <a href="https://github.com/ashimov/rke2-ansible/actions/workflows/lint.yml"><img src="https://github.com/ashimov/rke2-ansible/actions/workflows/lint.yml/badge.svg" alt="Lint"></a>
</p>

<p align="center">
  <a href="#quick-start">Quick Start</a> •
  <a href="#features">Features</a> •
  <a href="#documentation">Documentation</a> •
  <a href="./README.ru.md">Русский</a>
</p>

---

## Overview

This Ansible collection automates the deployment and management of [RKE2](https://docs.rke2.io/) (Rancher Kubernetes Engine 2) clusters. RKE2, also known as RKE Government, is Rancher's next-generation Kubernetes distribution focused on security and compliance.

> [!NOTE]
> This is version 1.0 of the refactored repository. Please review the [documentation](./docs/README.md) for configuration details.

## Features

- **Multi-OS Support**: Rocky Linux 8/9, RHEL 8/9, Ubuntu 22.04/24.04
- **Flexible Installation**: RPM packages or tarball-based air-gap deployments
- **Security Hardening**: Built-in CIS benchmark compliance and SELinux support
- **Modular Design**: Granular control with comprehensive Ansible tags
- **Production Ready**: Tested CI/CD pipelines with idempotent execution

## Requirements

| Component | Version |
|-----------|---------|
| Ansible | 2.17.0+ |
| Python | 3.10+ |

## Quick Start

### 1. Clone the Repository

```bash
git clone https://github.com/ashimov/rke2-ansible.git
cd rke2-ansible
```

### 2. Create Inventory

Create `inventory/hosts.yml`:

```yaml
---
rke2_cluster:
  children:
    rke2_servers:
      hosts:
        server1.example.com:
    rke2_agents:
      hosts:
        agent1.example.com:
        agent2.example.com:
```

### 3. Deploy Cluster

```bash
ansible-playbook site.yml -i inventory/hosts.yml -b
```

## Documentation

| Document | Description |
|----------|-------------|
| [Configuration Guide](./docs/README.md) | Complete configuration reference |
| [Air-Gap Installation](./docs/tarball_install.md) | Offline/tarball installation guide |
| [Development](./docs/development.md) | Contributing and development setup |

## Installation Methods

### Method 1: Clone Repository (Recommended)

```bash
git clone https://github.com/ashimov/rke2-ansible.git
```

### Method 2: Ansible Galaxy Collection

Add to `requirements.yml`:

```yaml
collections:
  - name: ashimov.rke2_ansible
    source: https://github.com/ashimov/rke2-ansible.git
    type: git
    version: main
```

Install:

```bash
ansible-galaxy collection install -r requirements.yml
```

Use in playbook:

```yaml
---
- name: Deploy RKE2 Cluster
  hosts: all
  roles:
    - role: ashimov.rke2_ansible.rke2
```

## Configuration Examples

### Basic Cluster with SELinux

```yaml
# inventory/group_vars/all.yml
---
cluster_rke2_config:
  selinux: true
```

### CIS Hardened Cluster

```yaml
# inventory/group_vars/rke2_servers.yml
---
group_rke2_config:
  profile: cis
```

### Specific RKE2 Version

```yaml
# inventory/group_vars/all.yml
---
rke2_install_version: v1.29.12+rke2r1
```

## Cluster Access

After deployment, `kubectl` is automatically configured for the root user on server nodes:

```bash
ssh root@server1.example.com
kubectl get nodes
```

## Uninstall

**RPM Installation:**

```bash
ansible -i inventory/hosts.yml all -a "/usr/bin/rke2-uninstall.sh" -b
```

**Tarball Installation:**

```bash
ansible -i inventory/hosts.yml all -a "/usr/local/bin/rke2-uninstall.sh" -b
```

> [!WARNING]
> Uninstalling RKE2 permanently deletes all cluster data.

## Known Issues

### fapolicyd on RHEL 8+

For systems with fapolicyd enabled, add rules before installation:

```bash
cat <<-EOF >> /etc/fapolicyd/rules.d/80-rke2.rules
allow perm=any all : dir=/var/lib/rancher/
allow perm=any all : dir=/opt/cni/
allow perm=any all : dir=/run/k3s/
allow perm=any all : dir=/var/lib/kubelet/
EOF

systemctl restart fapolicyd
```

## Support

- **Issues**: Addressed on a best-effort basis
- **Contributions**: Pull requests welcome

## License

See [LICENSE](./LICENSE) for details.
