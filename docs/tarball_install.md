# Air-Gap / Tarball Installation Guide

> **Language**: [English](./tarball_install.md) | [Русский](./tarball_install.ru.md)

## Table of Contents

- [Overview](#overview)
- [Prerequisites](#prerequisites)
- [Downloading Resources](#downloading-resources)
- [Configuration](#configuration)
  - [Tarball Installation](#tarball-installation)
  - [Container Images](#container-images)
- [Complete Example](#complete-example)

---

## Overview

RKE2 supports air-gapped deployments through two methods:

1. **Tarball Installation** — Deploy RKE2 binaries from local tarballs
2. **Private Registry** — Use a private container registry

This guide covers the tarball-based approach for fully disconnected environments.

> [!WARNING]
> For SELinux-enabled air-gapped systems, install the SELinux policy RPM before proceeding. See [RKE2 RPM Documentation](https://docs.rke2.io/install/methods/#rpm).

---

## Prerequisites

Download the following from [RKE2 Releases](https://github.com/rancher/rke2/releases):

| File | Description |
|------|-------------|
| `rke2.linux-amd64.tar.gz` | RKE2 binaries |
| `rke2-images.linux-amd64.tar.zst` | Core container images |
| `sha256sum-amd64.txt` | Checksums for verification |

**Optional (CNI-specific):**

| File | Description |
|------|-------------|
| `rke2-images-canal.linux-amd64.tar.zst` | Canal CNI images |
| `rke2-images-cilium.linux-amd64.tar.zst` | Cilium CNI images |
| `rke2-images-calico.linux-amd64.tar.zst` | Calico CNI images |

---

## Downloading Resources

```bash
# Set version
RKE2_VERSION="v1.29.12+rke2r1"

# Download binaries
wget "https://github.com/rancher/rke2/releases/download/${RKE2_VERSION}/rke2.linux-amd64.tar.gz"

# Download images
wget "https://github.com/rancher/rke2/releases/download/${RKE2_VERSION}/rke2-images.linux-amd64.tar.zst"

# Verify checksums
wget "https://github.com/rancher/rke2/releases/download/${RKE2_VERSION}/sha256sum-amd64.txt"
sha256sum -c sha256sum-amd64.txt --ignore-missing
```

---

## Configuration

### Tarball Installation

Three variables control tarball installation:

| Variable | Description |
|----------|-------------|
| `rke2_install_tarball_url` | URL to download tarball |
| `rke2_install_local_tarball_path` | Local path to tarball |
| `rke2_force_tarball_install` | Force tarball method (downloads from GitHub) |

> [!WARNING]
> Use only ONE of `rke2_install_tarball_url` OR `rke2_install_local_tarball_path`, not both.

**Example: Local tarball**

```yaml
# group_vars/all.yml
---
rke2_install_local_tarball_path: "{{ playbook_dir }}/files/rke2.linux-amd64.tar.gz"
```

**Example: Remote URL**

```yaml
# group_vars/all.yml
---
rke2_install_tarball_url: "https://internal-repo.example.com/rke2/rke2.linux-amd64.tar.gz"
```

**Example: Force tarball from GitHub**

```yaml
# group_vars/all.yml
---
rke2_force_tarball_install: true
rke2_install_version: v1.29.12+rke2r1
```

### Container Images

For air-gapped environments, provide pre-loaded container images:

| Variable | Description |
|----------|-------------|
| `rke2_images_urls` | List of URLs to download images |
| `rke2_images_local_tarball_path` | List of local image tarballs |

**Example: Local images**

```yaml
# group_vars/all.yml
---
rke2_images_local_tarball_path:
  - "{{ playbook_dir }}/files/rke2-images.linux-amd64.tar.zst"
  - "{{ playbook_dir }}/files/rke2-images-cilium.linux-amd64.tar.zst"
```

**Example: Remote URLs**

```yaml
# group_vars/all.yml
---
rke2_images_urls:
  - "https://internal-repo.example.com/rke2/rke2-images.linux-amd64.tar.zst"
```

---

## Complete Example

### Directory Structure

```text
inventory/
├── hosts.yml
├── group_vars/
│   └── all.yml
└── files/
    ├── rke2.linux-amd64.tar.gz
    └── rke2-images.linux-amd64.tar.zst
```

### hosts.yml

```yaml
---
rke2_cluster:
  children:
    rke2_servers:
      hosts:
        server1.example.com:
        server2.example.com:
        server3.example.com:
    rke2_agents:
      hosts:
        agent1.example.com:
        agent2.example.com:
```

### group_vars/all.yml

```yaml
---
# Tarball installation
rke2_install_local_tarball_path: "{{ playbook_dir }}/files/rke2.linux-amd64.tar.gz"

# Pre-loaded images
rke2_images_local_tarball_path:
  - "{{ playbook_dir }}/files/rke2-images.linux-amd64.tar.zst"

# Cluster configuration
cluster_rke2_config:
  selinux: true
```

### Deploy

```bash
ansible-playbook site.yml -i inventory/hosts.yml -b
```

---

## Troubleshooting

### Image Loading Issues

If images fail to load, verify:

1. File permissions on image tarballs
2. Sufficient disk space in `/var/lib/rancher/rke2/agent/images/`
3. Correct file format (`.tar.zst` or `.tar.gz`)

### SELinux Denials

For SELinux-related failures:

```bash
# Check for denials
ausearch -m avc -ts recent

# Install SELinux policy
yum install -y rke2-selinux
```

### Verify Installation

```bash
# Check RKE2 service
systemctl status rke2-server

# Verify images loaded
ls -la /var/lib/rancher/rke2/agent/images/

# Check containerd
/var/lib/rancher/rke2/bin/crictl images
```
