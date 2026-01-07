# RKE2 Ansible Configuration Guide

> **Language**: [English](./README.md) | [Русский](./README.ru.md)

## Table of Contents

- [RKE2 Ansible Configuration Guide](#rke2-ansible-configuration-guide)
  - [Table of Contents](#table-of-contents)
  - [Getting Started](#getting-started)
    - [Installation Methods](#installation-methods)
      - [Method 1: Clone Repository](#method-1-clone-repository)
      - [Method 2: Ansible Galaxy Collection](#method-2-ansible-galaxy-collection)
    - [Repository Structure](#repository-structure)
  - [Inventory Configuration](#inventory-configuration)
    - [Minimal Inventory](#minimal-inventory)
    - [Organizing Variables](#organizing-variables)
  - [RKE2 Configuration](#rke2-configuration)
    - [Configuration Hierarchy](#configuration-hierarchy)
    - [Specifying RKE2 Version](#specifying-rke2-version)
    - [Enabling SELinux](#enabling-selinux)
    - [CIS Hardening](#cis-hardening)
  - [Advanced Configuration](#advanced-configuration)
    - [Pod Security Admission](#pod-security-admission)
    - [Audit Policy](#audit-policy)
    - [Custom Manifests](#custom-manifests)
  - [Available Tags](#available-tags)
  - [Role Variables Reference](#role-variables-reference)
    - [Installation](#installation)
    - [Paths](#paths)
    - [Network](#network)
    - [Timeouts](#timeouts)
  - [Examples](#examples)

---

## Getting Started

### Installation Methods

#### Method 1: Clone Repository

The simplest approach — clone and customize:

```bash
git clone https://github.com/ashimov/rke2-ansible.git
cd rke2-ansible
cp -r docs/basic_sample_inventory inventory/my_cluster
```

#### Method 2: Ansible Galaxy Collection

Add to your `requirements.yml`:

```yaml
collections:
  - name: ashimov.rke2_ansible
    source: https://github.com/ashimov/rke2-ansible.git
    type: git
    version: main
```

Install and use:

```bash
ansible-galaxy collection install -r requirements.yml
```

```yaml
---
- name: Deploy RKE2
  hosts: all
  any_errors_fatal: true
  roles:
    - role: ashimov.rke2_ansible.rke2
```

### Repository Structure

```
rke2-ansible/
├── roles/
│   ├── rke2/                    # Main RKE2 role
│   │   ├── defaults/main.yml    # Default variables
│   │   ├── tasks/               # Task files
│   │   └── handlers/            # Handlers
│   └── testing/                 # Testing role
├── docs/
│   ├── basic_sample_inventory/  # Simple inventory example
│   └── advanced_sample_inventory/ # Full-featured example
├── site.yml                     # Main playbook
└── testing.yml                  # Testing playbook
```

---

## Inventory Configuration

### Minimal Inventory

The simplest working inventory:

```yaml
# inventory/hosts.yml
---
rke2_cluster:
  children:
    rke2_servers:
      hosts:
        server1.example.com:
    rke2_agents:
      hosts:
        agent1.example.com:
```

This deploys the latest stable RKE2 with default settings.

### Organizing Variables

For complex deployments, use `group_vars`:

```
inventory/
├── hosts.yml
└── group_vars/
    ├── all.yml           # Cluster-wide settings
    ├── rke2_servers.yml  # Server-specific settings
    └── rke2_agents.yml   # Agent-specific settings
```

**Example multi-cluster layout:**

```
inventory/
├── production/
│   ├── hosts.yml
│   └── group_vars/
│       ├── all.yml
│       └── rke2_servers.yml
└── staging/
    ├── hosts.yml
    └── group_vars/
        └── all.yml
```

---

## RKE2 Configuration

### Configuration Hierarchy

RKE2 configuration is merged from three levels (lower overrides higher):

| Level | Variable | Scope |
|-------|----------|-------|
| 1 | `cluster_rke2_config` | All nodes |
| 2 | `group_rke2_config` | Server or agent group |
| 3 | `host_rke2_config` | Individual host |

**Example:**

```yaml
# group_vars/all.yml - applies to all nodes
cluster_rke2_config:
  selinux: true
  write-kubeconfig-mode: "0640"

# group_vars/rke2_servers.yml - only servers
group_rke2_config:
  profile: cis
  tls-san:
    - "kubernetes.example.com"

# hosts.yml - specific host
rke2_servers:
  hosts:
    server1:
      host_rke2_config:
        node-label:
          - "topology.kubernetes.io/zone=us-east-1a"
```

### Specifying RKE2 Version

```yaml
# group_vars/all.yml
---
rke2_install_version: v1.29.12+rke2r1
```

Find available versions at [RKE2 Releases](https://github.com/rancher/rke2/releases).

### Enabling SELinux

```yaml
# group_vars/all.yml
---
cluster_rke2_config:
  selinux: true
```

See [RKE2 SELinux Documentation](https://docs.rke2.io/security/selinux) for details.

### CIS Hardening

```yaml
# group_vars/rke2_servers.yml
---
group_rke2_config:
  profile: cis
```

Available profiles: `cis`, `cis-1.23`

See [RKE2 Hardening Guide](https://docs.rke2.io/security/hardening_guide) for requirements.

---

## Advanced Configuration

### Pod Security Admission

**Step 1:** Create PSA config file:

```yaml
# files/pod-security-admission-config.yaml
apiVersion: pod-security.admission.config.k8s.io/v1
kind: PodSecurityConfiguration
defaults:
  enforce: "restricted"
  enforce-version: "latest"
  audit: "restricted"
  audit-version: "latest"
  warn: "restricted"
  warn-version: "latest"
exemptions:
  usernames: []
  runtimeClasses: []
  namespaces:
    - kube-system
    - cattle-system
```

**Step 2:** Configure the role:

```yaml
# group_vars/rke2_servers.yml
---
rke2_pod_security_admission_config_file_path: "{{ playbook_dir }}/files/pod-security-admission-config.yaml"

group_rke2_config:
  pod-security-admission-config-file: /etc/rancher/rke2/pod-security-admission-config.yaml
```

### Audit Policy

**Step 1:** Create audit policy:

```yaml
# files/audit-policy.yaml
apiVersion: audit.k8s.io/v1
kind: Policy
rules:
  - level: Metadata
    resources:
      - group: ""
        resources: ["secrets", "configmaps"]
  - level: RequestResponse
    resources:
      - group: ""
        resources: ["pods"]
```

**Step 2:** Configure the role:

```yaml
# group_vars/rke2_servers.yml
---
rke2_audit_policy_config_file_path: "{{ playbook_dir }}/files/audit-policy.yaml"

group_rke2_config:
  audit-policy-file: /etc/rancher/rke2/audit-policy.yaml
  kube-apiserver-arg:
    - audit-policy-file=/etc/rancher/rke2/audit-policy.yaml
    - audit-log-path=/var/lib/rancher/rke2/server/logs/audit.log
```

### Custom Manifests

Deploy manifests automatically using RKE2's HelmChart/HelmChartConfig resources.

**Pre-deploy manifests** (before RKE2 starts):

```yaml
# group_vars/rke2_servers.yml
---
rke2_manifest_config_directory: "{{ playbook_dir }}/manifests/pre-deploy/"

group_rke2_config:
  cni:
    - cilium
  disable-kube-proxy: true
```

```yaml
# manifests/pre-deploy/cilium.yaml
---
apiVersion: helm.cattle.io/v1
kind: HelmChartConfig
metadata:
  name: rke2-cilium
  namespace: kube-system
spec:
  valuesContent: |-
    kubeProxyReplacement: true
    k8sServiceHost: 127.0.0.1
    k8sServicePort: 6443
```

**Post-deploy manifests** (after RKE2 starts):

```yaml
# group_vars/rke2_servers.yml
---
rke2_manifest_config_post_run_directory: "{{ playbook_dir }}/manifests/post-deploy/"
```

```yaml
# manifests/post-deploy/cert-manager.yaml
---
apiVersion: helm.cattle.io/v1
kind: HelmChart
metadata:
  name: cert-manager
  namespace: kube-system
spec:
  repo: https://charts.jetstack.io
  chart: cert-manager
  version: v1.16.2
  targetNamespace: cert-manager
  createNamespace: true
  valuesContent: |-
    crds:
      enabled: true
```

---

## Available Tags

Control execution with `--tags` or `--skip-tags`:

| Tag | Description |
|-----|-------------|
| `always` | Always runs |
| `validation` | Pre-flight inventory checks |
| `facts` | System fact gathering |
| `prereqs` | OS prerequisites |
| `install` | Installation tasks |
| `tarball` | Tarball installation |
| `rpm` | RPM installation |
| `airgap` | Air-gap image handling |
| `config` | Configuration tasks |
| `hardening` | CIS hardening |
| `bootstrap` | First server bootstrap |
| `join` | Node join operations |
| `token` | Token management |
| `post-install` | Post-installation tasks |
| `utilities` | CLI tools setup |
| `manifests` | Manifest deployment |

**Examples:**

```bash
# Only install, skip configuration
ansible-playbook site.yml -i inventory/ --tags install

# Skip hardening tasks
ansible-playbook site.yml -i inventory/ --skip-tags hardening
```

---

## Role Variables Reference

### Installation

| Variable | Default | Description |
|----------|---------|-------------|
| `rke2_channel` | `stable` | Release channel |
| `rke2_install_version` | `""` | Specific version (e.g., `v1.29.12+rke2r1`) |
| `rke2_force_tarball_install` | `false` | Force tarball installation |

### Paths

| Variable | Default | Description |
|----------|---------|-------------|
| `rke2_config_dir` | `/etc/rancher/rke2` | Config directory |
| `rke2_data_dir` | `/var/lib/rancher/rke2` | Data directory |
| `rke2_bin_path` | `/var/lib/rancher/rke2/bin` | Binary path |

### Network

| Variable | Default | Description |
|----------|---------|-------------|
| `rke2_api_port` | `6443` | Kubernetes API port |
| `rke2_supervisor_port` | `9345` | Supervisor API port |
| `rke2_https_port` | `443` | HTTPS ingress port (for iptables rules) |
| `rke2_add_iptables_rules` | `false` | Auto-add iptables rules |

### Timeouts

| Variable | Default | Description |
|----------|---------|-------------|
| `rke2_api_wait_timeout` | `300` | API server wait timeout (sec) |
| `rke2_node_ready_retries` | `30` | Node ready check retries |
| `rke2_node_ready_delay` | `10` | Delay between retries (sec) |

---

## Examples

See the sample inventories in this directory:

- **[basic_sample_inventory](./basic_sample_inventory/)** — Minimal configuration
- **[advanced_sample_inventory](./advanced_sample_inventory/)** — Full-featured with CIS, PSA, manifests
- **[tarball_sample_inventory](./tarball_sample_inventory/)** — Air-gap installation
