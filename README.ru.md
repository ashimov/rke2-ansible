<h1 align="center">RKE2 Ansible Collection</h1>

<p align="center">
  <strong>Автоматизация развёртывания RKE2 Kubernetes кластеров с помощью Ansible</strong>
</p>

<p align="center">
  <a href="https://github.com/ashimov/rke2-ansible/actions/workflows/lint.yml"><img src="https://github.com/ashimov/rke2-ansible/actions/workflows/lint.yml/badge.svg" alt="Lint"></a>
  <a href="https://github.com/ashimov/rke2-ansible/actions/workflows/release.yml"><img src="https://github.com/ashimov/rke2-ansible/actions/workflows/release.yml/badge.svg" alt="Release"></a>
  <a href="https://github.com/ashimov/rke2-ansible/releases"><img src="https://img.shields.io/github/v/release/ashimov/rke2-ansible" alt="GitHub Release"></a>
  <a href="./LICENSE"><img src="https://img.shields.io/github/license/ashimov/rke2-ansible" alt="License"></a>
</p>

<p align="center">
  <img src="https://img.shields.io/badge/ansible-%3E%3D2.17-blue.svg" alt="Ansible">
  <img src="https://img.shields.io/badge/python-%3E%3D3.10-blue.svg" alt="Python">
  <a href="https://github.com/pre-commit/pre-commit"><img src="https://img.shields.io/badge/pre--commit-enabled-brightgreen?logo=pre-commit" alt="pre-commit"></a>
  <a href="./CONTRIBUTING.md"><img src="https://img.shields.io/badge/contributions-welcome-brightgreen.svg" alt="Contributions Welcome"></a>
</p>

<p align="center">
  <a href="#быстрый-старт">Быстрый старт</a> •
  <a href="#возможности">Возможности</a> •
  <a href="#документация">Документация</a> •
  <a href="./README.md">English</a>
</p>

---

## Обзор

Эта Ansible коллекция автоматизирует развёртывание и управление кластерами [RKE2](https://docs.rke2.io/) (Rancher Kubernetes Engine 2). RKE2, также известный как RKE Government — это Kubernetes дистрибутив нового поколения от Rancher, ориентированный на безопасность и соответствие стандартам.

> [!NOTE]
> Это версия 1.0 переработанного репозитория. Ознакомьтесь с [документацией](./docs/README.ru.md) для деталей конфигурации.

## Возможности

- **Мульти-ОС поддержка**: Rocky Linux 8/9, RHEL 8/9, Ubuntu 22.04/24.04
- **Гибкая установка**: RPM пакеты или tarball для air-gap развёртываний
- **Усиление безопасности**: Встроенная поддержка CIS benchmark и SELinux
- **Модульный дизайн**: Гранулярный контроль через Ansible теги
- **Production Ready**: Протестированные CI/CD пайплайны с идемпотентным выполнением

## Требования

| Компонент | Версия |
|-----------|--------|
| Ansible   | 2.17.0+ |
| Python    | 3.10+   |

## Быстрый старт

### 1. Клонирование репозитория

```bash
git clone https://github.com/ashimov/rke2-ansible.git
cd rke2-ansible
```

### 2. Создание инвентаря

Создайте `inventory/hosts.yml`:

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

### 3. Развёртывание кластера

```bash
ansible-playbook site.yml -i inventory/hosts.yml -b
```

## Документация

| Документ | Описание |
|----------|----------|
| [Руководство по конфигурации](./docs/README.ru.md) | Полный справочник по настройке |
| [HA кластер](./docs/ha-cluster.ru.md) | Конфигурация высокодоступного кластера |
| [Обновление кластера](./docs/upgrade.ru.md) | Процедуры обновления |
| [Air-Gap установка](./docs/tarball_install.ru.md) | Установка в изолированных средах |
| [Резервное копирование](./docs/backup-restore.ru.md) | Бэкап и аварийное восстановление |
| [Мониторинг](./docs/monitoring.md) | Интеграция с Prometheus/Grafana |
| [Cloud-Init](./docs/cloud-init.md) | Шаблоны подготовки узлов |
| [Устранение неполадок](./docs/troubleshooting.ru.md) | Типичные проблемы и решения |
| [Разработка](./docs/development.ru.md) | Участие в разработке |

## Методы установки

### Метод 1: Клонирование репозитория (рекомендуется)

```bash
git clone https://github.com/ashimov/rke2-ansible.git
```

### Метод 2: Ansible Galaxy Collection

Добавьте в `requirements.yml`:

```yaml
collections:
  - name: ashimov.rke2_ansible
    source: https://github.com/ashimov/rke2-ansible.git
    type: git
    version: main
```

Установите:

```bash
ansible-galaxy collection install -r requirements.yml
```

Используйте в плейбуке:

```yaml
---
- name: Развёртывание RKE2 кластера
  hosts: all
  roles:
    - role: ashimov.rke2_ansible.rke2
```

## Примеры конфигурации

### Базовый кластер с SELinux

```yaml
# inventory/group_vars/all.yml
---
cluster_rke2_config:
  selinux: true
```

### Кластер с CIS Hardening

```yaml
# inventory/group_vars/rke2_servers.yml
---
group_rke2_config:
  profile: cis
```

### Указание версии RKE2

```yaml
# inventory/group_vars/all.yml
---
rke2_install_version: v1.29.12+rke2r1
```

## Доступ к кластеру

После развёртывания `kubectl` автоматически настроен для пользователя root на серверных нодах:

```bash
ssh root@server1.example.com
kubectl get nodes
```

## Удаление

**RPM установка:**

```bash
ansible -i inventory/hosts.yml all -a "/usr/bin/rke2-uninstall.sh" -b
```

**Tarball установка:**

```bash
ansible -i inventory/hosts.yml all -a "/usr/local/bin/rke2-uninstall.sh" -b
```

> [!WARNING]
> Удаление RKE2 безвозвратно удаляет все данные кластера.

## Известные проблемы

### fapolicyd на RHEL 8+

Для систем с включённым fapolicyd добавьте правила перед установкой:

```bash
cat <<-EOF >> /etc/fapolicyd/rules.d/80-rke2.rules
allow perm=any all : dir=/var/lib/rancher/
allow perm=any all : dir=/opt/cni/
allow perm=any all : dir=/run/k3s/
allow perm=any all : dir=/var/lib/kubelet/
EOF

systemctl restart fapolicyd
```

## Поддержка

- **Проблемы**: Решаются по мере возможности
- **Вклад**: Pull requests приветствуются

## Лицензия

Подробности в файле [LICENSE](./LICENSE).
