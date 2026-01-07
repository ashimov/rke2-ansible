# Development Guide

> **Language**: [English](./development.md) | [Русский](./development.ru.md)

## Table of Contents

- [Prerequisites](#prerequisites)
- [Setting Up Development Environment](#setting-up-development-environment)
- [Code Quality](#code-quality)
- [Testing](#testing)
- [Contributing](#contributing)

---

## Prerequisites

### Required Tools

| Tool | Version | Purpose |
|------|---------|---------|
| Python | 3.10+ | Runtime |
| Ansible | 2.17+ | Automation |
| ansible-lint | Latest | Code linting |
| yamllint | Latest | YAML validation |

### Installation

```bash
# Create virtual environment
python3 -m venv .venv
source .venv/bin/activate

# Install dependencies
pip install ansible ansible-lint yamllint pytest-testinfra
```

---

## Setting Up Development Environment

### Clone Repository

```bash
git clone https://github.com/ashimov/rke2-ansible.git
cd rke2-ansible
```

### Install Ansible Dependencies

```bash
ansible-galaxy collection install -r requirements.yml
```

### Project Structure

```
rke2-ansible/
├── roles/
│   ├── rke2/
│   │   ├── defaults/main.yml    # Default variables
│   │   ├── tasks/
│   │   │   ├── main.yml         # Main orchestration
│   │   │   ├── common/          # Reusable tasks
│   │   │   ├── config.yml       # Configuration
│   │   │   └── ...
│   │   └── handlers/main.yml    # Event handlers
│   └── testing/                 # Testing role
├── testing/                     # Integration test infrastructure
├── .github/workflows/           # CI/CD pipelines
└── docs/                        # Documentation
```

---

## Code Quality

### Linting

Run all linters:

```bash
# Ansible lint (production profile)
ansible-lint --profile=production ./roles

# YAML lint
yamllint .
```

### Code Standards

1. **Task Names**: Use descriptive names in sentence case
   ```yaml
   - name: Create configuration directory
   ```

2. **Variable Naming**: Use `snake_case` with `rke2_` prefix
   ```yaml
   rke2_config_dir: "/etc/rancher/rke2"
   ```

3. **Internal Variables**: Prefix with underscore
   ```yaml
   register: _api_status
   ```

4. **Module Usage**: Prefer `ansible.builtin.systemd` over `ansible.builtin.service`

5. **Comments**: Use section headers for organization
   ```yaml
   # =============================================================================
   # Section Name
   # =============================================================================
   ```

6. **Tags**: Add appropriate tags to all task includes
   ```yaml
   - name: Configure RKE2
     ansible.builtin.include_tasks: configure_rke2.yml
     tags:
       - config
       - hardening
   ```

---

## Testing

### Running Tests

**Ansible Tests:**

```bash
ansible-playbook testing.yml -i inventory/hosts.yml --skip-tags troubleshooting
```

**Python Tests (testinfra):**

```bash
pytest --hosts=rke2_servers \
  --ansible-inventory=inventory/hosts.yml \
  --force-ansible \
  --connection=ansible \
  --sudo \
  testing/basic_server_tests.py
```

### CI/CD Pipeline

The project uses GitHub Actions for CI/CD:

| Workflow | Trigger | Description |
|----------|---------|-------------|
| `lint.yml` | Push, PR | ansible-lint, yamllint, vale |
| `ubuntu.yml` | PR | Integration tests on Ubuntu |
| `rocky.yml` | PR | Integration tests on Rocky Linux |

### Local Integration Testing

Requirements:
- AWS credentials configured
- Terraform installed

```bash
cd testing/
terraform init
terraform apply -var "GITHUB_RUN_ID=local-test" -var "os=rocky9"
```

---

## Contributing

### Pull Request Process

1. **Fork** the repository
2. **Create** a feature branch: `git checkout -b feature/my-feature`
3. **Make** your changes
4. **Test** your changes locally
5. **Lint** your code: `ansible-lint --profile=production ./roles`
6. **Commit** with clear messages
7. **Push** to your fork
8. **Create** a Pull Request

### Commit Message Format

```
type: Short description

Longer description if needed.

Fixes #123
```

Types: `feat`, `fix`, `docs`, `refactor`, `test`, `chore`

### Code Review Checklist

- [ ] Code passes `ansible-lint --profile=production`
- [ ] Code passes `yamllint`
- [ ] All task names are descriptive
- [ ] No hardcoded paths (use variables)
- [ ] Appropriate tags added
- [ ] Documentation updated if needed
