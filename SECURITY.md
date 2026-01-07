# Security Policy

## Supported Versions

We release patches for security vulnerabilities in the following versions:

| Version | Supported          |
| ------- | ------------------ |
| 1.x.x   | :white_check_mark: |
| < 1.0   | :x:                |

## Reporting a Vulnerability

We take security seriously. If you discover a security vulnerability, please report it responsibly.

### How to Report

**Please do NOT report security vulnerabilities through public GitHub issues.**

Instead, please report them via email or private communication:

1. **Email**: Send details to the repository maintainer (check profile for contact)
2. **Private Disclosure**: Use GitHub's private vulnerability reporting feature

### What to Include

Please include the following information in your report:

- Type of vulnerability (e.g., credential exposure, privilege escalation)
- Full paths of source files related to the vulnerability
- Step-by-step instructions to reproduce the issue
- Proof-of-concept or exploit code (if possible)
- Impact assessment

### Response Timeline

- **Initial Response**: Within 48 hours
- **Status Update**: Within 7 days
- **Resolution Target**: Within 30 days (depending on complexity)

### What to Expect

1. Acknowledgment of your report
2. Assessment of the vulnerability
3. Development of a fix
4. Coordinated disclosure

## Security Best Practices

When using this Ansible collection, follow these security recommendations:

### Credentials and Secrets

- **Never commit secrets** to version control
- Use Ansible Vault for sensitive variables
- Rotate tokens and credentials regularly
- Use `no_log: true` for tasks handling sensitive data

```yaml
# Example: Using Ansible Vault
cluster_rke2_config:
  token: "{{ vault_rke2_token }}"
```

### Network Security

- Use private networks for cluster communication
- Configure firewalls to restrict access to required ports only
- Enable TLS for all external communications
- Use network policies in Kubernetes

### RKE2 Hardening

- Enable CIS benchmark profile where applicable
- Configure SELinux in enforcing mode
- Review and restrict RBAC permissions
- Enable audit logging

```yaml
# Example: CIS hardening configuration
group_rke2_config:
  profile: cis
```

### Access Control

- Use SSH keys instead of passwords
- Implement least-privilege access
- Regularly audit user access
- Use separate service accounts for automation

### Backup Security

- Encrypt etcd backups
- Store backups in secure, separate location
- Test backup restoration regularly
- Implement backup retention policies

## Known Security Considerations

### Air-Gap Installations

For air-gap environments:

- Verify checksums of all downloaded artifacts
- Use trusted internal registries
- Scan container images for vulnerabilities
- Keep offline mirrors updated

### Token Handling

The RKE2 cluster token is sensitive:

- Generated tokens are stored with restricted permissions
- Token files should not be world-readable
- Rotate tokens when nodes are decommissioned

### File Permissions

This collection sets appropriate permissions:

- Configuration files: 0600 or 0640
- Directories: 0700 or 0750
- Executables: 0755

## Security Updates

Security updates will be released as patch versions. We recommend:

1. Subscribe to repository releases
2. Review changelogs for security fixes
3. Update promptly when security patches are available
4. Test updates in non-production environments first

## Acknowledgments

We appreciate responsible disclosure of security issues. Contributors who report valid vulnerabilities will be acknowledged (unless they prefer to remain anonymous).
