# Cloud-Init Examples

This guide provides cloud-init templates for preparing nodes before running the RKE2 Ansible playbook.

## Table of Contents

- [Overview](#overview)
- [Base Template](#base-template)
- [Ubuntu Template](#ubuntu-template)
- [Rocky Linux Template](#rocky-linux-template)
- [AWS EC2 Examples](#aws-ec2-examples)
- [Advanced Configuration](#advanced-configuration)

## Overview

Cloud-init automates initial node setup:
- Configure SSH access
- Install prerequisites
- Set up networking
- Configure storage
- Prepare system for RKE2

## Base Template

Minimal cloud-init for all distributions:

```yaml
#cloud-config

# Set hostname
hostname: ${hostname}
fqdn: ${hostname}.${domain}
manage_etc_hosts: true

# Configure SSH
ssh_authorized_keys:
  - ssh-rsa AAAA... your-key-here

# Disable password authentication
ssh_pwauth: false

# Update and install packages
package_update: true
package_upgrade: true

# Create ansible user
users:
  - name: ansible
    groups: [sudo, wheel]
    shell: /bin/bash
    sudo: ['ALL=(ALL) NOPASSWD:ALL']
    ssh_authorized_keys:
      - ssh-rsa AAAA... your-ansible-key

# Configure timezone
timezone: UTC

# Final message
final_message: "Cloud-init completed after $UPTIME seconds"
```

## Ubuntu Template

Full template for Ubuntu 22.04/24.04:

```yaml
#cloud-config

hostname: ${hostname}
fqdn: ${hostname}.${domain}
manage_etc_hosts: true

users:
  - name: ansible
    groups: [sudo]
    shell: /bin/bash
    sudo: ['ALL=(ALL) NOPASSWD:ALL']
    ssh_authorized_keys:
      - ${ssh_public_key}

ssh_pwauth: false

package_update: true
package_upgrade: true

packages:
  - curl
  - wget
  - vim
  - htop
  - iotop
  - net-tools
  - python3
  - python3-pip
  - open-iscsi
  - nfs-common

# Disable swap (required for Kubernetes)
swap:
  filename: /swap.img
  size: 0

# Configure kernel modules
write_files:
  - path: /etc/modules-load.d/k8s.conf
    content: |
      overlay
      br_netfilter

  - path: /etc/sysctl.d/k8s.conf
    content: |
      net.bridge.bridge-nf-call-iptables  = 1
      net.bridge.bridge-nf-call-ip6tables = 1
      net.ipv4.ip_forward                 = 1

# Run commands
runcmd:
  # Disable swap
  - swapoff -a
  - sed -i '/swap/d' /etc/fstab

  # Load kernel modules
  - modprobe overlay
  - modprobe br_netfilter

  # Apply sysctl
  - sysctl --system

  # Enable and start iscsid (for Longhorn/CSI)
  - systemctl enable --now iscsid

  # Signal ready
  - touch /var/run/cloud-init-complete

final_message: "Cloud-init completed in $UPTIME seconds"
```

## Rocky Linux Template

Full template for Rocky Linux 8/9:

```yaml
#cloud-config

hostname: ${hostname}
fqdn: ${hostname}.${domain}
manage_etc_hosts: true

users:
  - name: ansible
    groups: [wheel]
    shell: /bin/bash
    sudo: ['ALL=(ALL) NOPASSWD:ALL']
    ssh_authorized_keys:
      - ${ssh_public_key}

ssh_pwauth: false

package_update: true
package_upgrade: true

packages:
  - curl
  - wget
  - vim
  - htop
  - iotop
  - net-tools
  - python3
  - python3-pip
  - iscsi-initiator-utils
  - nfs-utils
  - container-selinux
  - policycoreutils-python-utils

write_files:
  - path: /etc/modules-load.d/k8s.conf
    content: |
      overlay
      br_netfilter

  - path: /etc/sysctl.d/k8s.conf
    content: |
      net.bridge.bridge-nf-call-iptables  = 1
      net.bridge.bridge-nf-call-ip6tables = 1
      net.ipv4.ip_forward                 = 1

  # fapolicyd rules for RKE2 (if fapolicyd is enabled)
  - path: /etc/fapolicyd/rules.d/80-rke2.rules
    content: |
      allow perm=any all : dir=/var/lib/rancher/
      allow perm=any all : dir=/opt/cni/
      allow perm=any all : dir=/run/k3s/
      allow perm=any all : dir=/var/lib/kubelet/

runcmd:
  # Disable swap
  - swapoff -a
  - sed -i '/swap/d' /etc/fstab

  # Load kernel modules
  - modprobe overlay
  - modprobe br_netfilter

  # Apply sysctl
  - sysctl --system

  # Configure SELinux for RKE2
  - semanage fcontext -a -t container_var_lib_t "/var/lib/rancher/rke2(/.*)?"
  - semanage fcontext -a -t container_runtime_exec_t "/var/lib/rancher/rke2/bin(/.*)?"

  # Enable and start iscsid
  - systemctl enable --now iscsid

  # Restart fapolicyd if running
  - systemctl restart fapolicyd || true

  # Signal ready
  - touch /var/run/cloud-init-complete

final_message: "Cloud-init completed in $UPTIME seconds"
```

## AWS EC2 Examples

### Terraform with cloud-init

```hcl
# main.tf

locals {
  cloud_init_server = templatefile("${path.module}/cloud-init-server.yaml", {
    hostname       = "server-${count.index + 1}"
    domain         = "example.com"
    ssh_public_key = var.ssh_public_key
  })
}

resource "aws_instance" "rke2_server" {
  count = 3

  ami           = data.aws_ami.ubuntu.id
  instance_type = "t3.xlarge"
  key_name      = aws_key_pair.deployer.key_name

  user_data = local.cloud_init_server

  vpc_security_group_ids = [aws_security_group.rke2.id]
  subnet_id              = aws_subnet.private[count.index].id

  root_block_device {
    volume_size = 100
    volume_type = "gp3"
  }

  tags = {
    Name = "rke2-server-${count.index + 1}"
    Role = "server"
  }
}

resource "aws_instance" "rke2_agent" {
  count = 3

  ami           = data.aws_ami.ubuntu.id
  instance_type = "t3.xlarge"
  key_name      = aws_key_pair.deployer.key_name

  user_data = templatefile("${path.module}/cloud-init-agent.yaml", {
    hostname       = "agent-${count.index + 1}"
    domain         = "example.com"
    ssh_public_key = var.ssh_public_key
  })

  vpc_security_group_ids = [aws_security_group.rke2.id]
  subnet_id              = aws_subnet.private[count.index].id

  root_block_device {
    volume_size = 200
    volume_type = "gp3"
  }

  tags = {
    Name = "rke2-agent-${count.index + 1}"
    Role = "agent"
  }
}
```

### AWS Launch Template

```hcl
resource "aws_launch_template" "rke2_node" {
  name_prefix   = "rke2-node-"
  image_id      = data.aws_ami.ubuntu.id
  instance_type = "t3.xlarge"

  user_data = base64encode(templatefile("${path.module}/cloud-init.yaml", {
    ssh_public_key = var.ssh_public_key
  }))

  block_device_mappings {
    device_name = "/dev/sda1"
    ebs {
      volume_size           = 100
      volume_type           = "gp3"
      delete_on_termination = true
      encrypted             = true
    }
  }

  network_interfaces {
    associate_public_ip_address = false
    security_groups             = [aws_security_group.rke2.id]
  }

  tag_specifications {
    resource_type = "instance"
    tags = {
      Environment = "production"
      ManagedBy   = "terraform"
    }
  }
}
```

## Advanced Configuration

### Multi-disk Setup

```yaml
#cloud-config

# Configure additional disk for data
disk_setup:
  /dev/nvme1n1:
    table_type: gpt
    layout: true
    overwrite: false

fs_setup:
  - label: data
    filesystem: xfs
    device: /dev/nvme1n1p1

mounts:
  - ["/dev/nvme1n1p1", "/var/lib/rancher", "xfs", "defaults,noatime", "0", "2"]
```

### Network Configuration

```yaml
#cloud-config

# Static IP configuration
network:
  version: 2
  ethernets:
    eth0:
      dhcp4: false
      addresses:
        - 10.0.0.10/24
      gateway4: 10.0.0.1
      nameservers:
        addresses: [10.0.0.2, 8.8.8.8]
```

### Private Registry Configuration

```yaml
#cloud-config

write_files:
  - path: /etc/rancher/rke2/registries.yaml
    content: |
      mirrors:
        docker.io:
          endpoint:
            - "https://registry.example.com"
      configs:
        "registry.example.com":
          auth:
            username: ${registry_username}
            password: ${registry_password}
```

### Wait for Cloud-Init in Ansible

Add to your playbook to wait for cloud-init:

```yaml
# site.yml
- name: Wait for cloud-init to complete
  hosts: all
  gather_facts: false
  tasks:
    - name: Wait for cloud-init
      ansible.builtin.wait_for:
        path: /var/run/cloud-init-complete
        timeout: 600

- name: Deploy RKE2
  hosts: all
  become: true
  roles:
    - role: rke2
```
