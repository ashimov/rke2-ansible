# Contributing to RKE2 Ansible Collection

Thank you for your interest in contributing! This document provides guidelines and instructions for contributing to this project.

## Table of Contents

- [Code of Conduct](#code-of-conduct)
- [Getting Started](#getting-started)
- [Development Setup](#development-setup)
- [Making Changes](#making-changes)
- [Coding Standards](#coding-standards)
- [Testing](#testing)
- [Submitting Changes](#submitting-changes)
- [Documentation](#documentation)

## Code of Conduct

Please be respectful and considerate in all interactions. We aim to foster an inclusive and welcoming community.

## Getting Started

1. Fork the repository
2. Clone your fork locally
3. Set up the development environment
4. Create a branch for your changes
5. Make your changes and test them
6. Submit a pull request

## Development Setup

### Prerequisites

- Python 3.10+
- Ansible 2.17.0+
- pre-commit

### Environment Setup

```bash
# Clone your fork
git clone https://github.com/YOUR_USERNAME/rke2-ansible.git
cd rke2-ansible

# Create virtual environment
python3 -m venv venv
source venv/bin/activate

# Install dependencies
pip install ansible ansible-lint yamllint pre-commit

# Install pre-commit hooks
pre-commit install

# Verify setup
ansible-lint --version
yamllint --version
pre-commit --version
```

### Pre-commit Hooks

This project uses pre-commit hooks to ensure code quality. The hooks run automatically on `git commit`, but you can also run them manually:

```bash
# Run all hooks on all files
pre-commit run --all-files

# Run specific hook
pre-commit run ansible-lint --all-files
```

## Making Changes

### Branch Naming

Use descriptive branch names:

- `feature/add-xyz-support` - New features
- `fix/issue-123-description` - Bug fixes
- `docs/update-readme` - Documentation updates
- `refactor/improve-xyz` - Code refactoring

### Commit Messages

Write clear, concise commit messages:

```text
Short summary (50 chars or less)

More detailed explanation if needed. Wrap at 72 characters.
Explain the problem this commit solves and why.

- Bullet points are okay
- Use present tense ("Add feature" not "Added feature")
```

## Coding Standards

### Ansible Best Practices

1. **Use fully qualified collection names (FQCN)**:

   ```yaml
   # Good
   - name: Install package
     ansible.builtin.package:
       name: vim

   # Avoid
   - name: Install package
     package:
       name: vim
   ```

2. **Always name tasks**:

   ```yaml
   # Good
   - name: Configure RKE2 service
     ansible.builtin.template:
       src: config.yaml.j2
       dest: /etc/rancher/rke2/config.yaml

   # Avoid
   - ansible.builtin.template:
       src: config.yaml.j2
       dest: /etc/rancher/rke2/config.yaml
   ```

3. **Use meaningful variable names**:

   ```yaml
   # Good
   rke2_config_dir: /etc/rancher/rke2

   # Avoid
   dir: /etc/rancher/rke2
   ```

4. **Register variables with underscore prefix for internal use**:

   ```yaml
   - name: Check service status
     ansible.builtin.command: systemctl is-active rke2-server
     register: _rke2_service_status
   ```

5. **Use `no_log: true` for sensitive data**:

   ```yaml
   - name: Configure token
     ansible.builtin.template:
       src: token.j2
       dest: /etc/rancher/rke2/token
     no_log: true
   ```

### YAML Style

- Use 2 spaces for indentation
- No trailing whitespace
- Single newline at end of file
- Maximum line length: 160 characters

### File Organization

```text
roles/rke2/
├── defaults/
│   └── main.yml          # Default variables
├── tasks/
│   ├── main.yml          # Main task entry point
│   └── *.yml             # Task files
├── templates/
│   └── *.j2              # Jinja2 templates
├── handlers/
│   └── main.yml          # Handlers
└── vars/
    └── main.yml          # Role variables
```

## Testing

### Local Testing

Before submitting a PR, test your changes locally:

```bash
# Run linters
ansible-lint --profile=production ./roles
yamllint -c .yamllint.yml .

# Run pre-commit
pre-commit run --all-files
```

### Test Environment

For testing deployments, you can use:

- Local VMs (Vagrant, libvirt)
- Cloud instances (AWS, GCP, Azure)
- Container-based testing

### Test Checklist

- [ ] Changes work on Rocky Linux 8/9
- [ ] Changes work on Ubuntu 22.04/24.04
- [ ] RPM installation method works
- [ ] Tarball installation method works
- [ ] Playbook runs are idempotent
- [ ] No ansible-lint warnings
- [ ] No yamllint errors

## Submitting Changes

### Pull Request Process

1. Update documentation if needed
2. Ensure all tests pass
3. Update CHANGELOG.md if applicable
4. Create a pull request with a clear description
5. Link any related issues

### PR Requirements

- Clear description of changes
- Tests pass (CI checks)
- Documentation updated (if applicable)
- Follows coding standards
- Signed commits (recommended)

### Review Process

1. Maintainers will review your PR
2. Address any feedback or requested changes
3. Once approved, your PR will be merged

## Documentation

### When to Update Documentation

- Adding new features
- Changing existing behavior
- Adding new configuration options
- Fixing unclear documentation

### Documentation Style

- Use clear, concise language
- Include code examples where helpful
- Keep both English and Russian docs in sync
- Test any code examples

### Documentation Files

| File | Purpose |
|------|---------|
| `README.md` | Project overview (English) |
| `README.ru.md` | Project overview (Russian) |
| `docs/*.md` | Detailed documentation |
| `docs/*.ru.md` | Russian translations |

## Questions?

If you have questions about contributing, feel free to:

- Open a [Question issue](https://github.com/ashimov/rke2-ansible/issues/new?template=question.yml)
- Check existing [discussions and issues](https://github.com/ashimov/rke2-ansible/issues)

Thank you for contributing!
