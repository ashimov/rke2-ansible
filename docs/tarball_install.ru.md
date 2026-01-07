# Air-Gap / Tarball установка

> **Язык**: [English](./tarball_install.md) | [Русский](./tarball_install.ru.md)

## Содержание

- [Обзор](#обзор)
- [Предварительные требования](#предварительные-требования)
- [Скачивание ресурсов](#скачивание-ресурсов)
- [Конфигурация](#конфигурация)
  - [Установка через Tarball](#установка-через-tarball)
  - [Образы контейнеров](#образы-контейнеров)
- [Полный пример](#полный-пример)

---

## Обзор

RKE2 поддерживает развёртывание в изолированных средах двумя методами:

1. **Установка через Tarball** — Развёртывание бинарников RKE2 из локальных архивов
2. **Приватный Registry** — Использование приватного реестра контейнеров

Это руководство описывает подход на основе tarball для полностью изолированных сред.

> [!WARNING]
> Для систем с SELinux в изолированной среде установите RPM-пакет политики SELinux перед началом. См. [документацию RKE2 RPM](https://docs.rke2.io/install/methods/#rpm).

---

## Предварительные требования

Скачайте следующие файлы со страницы [RKE2 Releases](https://github.com/rancher/rke2/releases):

| Файл | Описание |
|------|----------|
| `rke2.linux-amd64.tar.gz` | Бинарники RKE2 |
| `rke2-images.linux-amd64.tar.zst` | Основные образы контейнеров |
| `sha256sum-amd64.txt` | Контрольные суммы для проверки |

**Опционально (для конкретного CNI):**

| Файл | Описание |
|------|----------|
| `rke2-images-canal.linux-amd64.tar.zst` | Образы Canal CNI |
| `rke2-images-cilium.linux-amd64.tar.zst` | Образы Cilium CNI |
| `rke2-images-calico.linux-amd64.tar.zst` | Образы Calico CNI |

---

## Скачивание ресурсов

```bash
# Установка версии
RKE2_VERSION="v1.29.12+rke2r1"

# Скачивание бинарников
wget "https://github.com/rancher/rke2/releases/download/${RKE2_VERSION}/rke2.linux-amd64.tar.gz"

# Скачивание образов
wget "https://github.com/rancher/rke2/releases/download/${RKE2_VERSION}/rke2-images.linux-amd64.tar.zst"

# Проверка контрольных сумм
wget "https://github.com/rancher/rke2/releases/download/${RKE2_VERSION}/sha256sum-amd64.txt"
sha256sum -c sha256sum-amd64.txt --ignore-missing
```

---

## Конфигурация

### Установка через Tarball

Три переменные управляют установкой через tarball:

| Переменная | Описание |
|------------|----------|
| `rke2_install_tarball_url` | URL для скачивания tarball |
| `rke2_install_local_tarball_path` | Локальный путь к tarball |
| `rke2_force_tarball_install` | Принудительный метод tarball (скачивает с GitHub) |

> [!WARNING]
> Используйте только ОДНУ из переменных: `rke2_install_tarball_url` ИЛИ `rke2_install_local_tarball_path`, не обе.

**Пример: Локальный tarball**

```yaml
# group_vars/all.yml
---
rke2_install_local_tarball_path: "{{ playbook_dir }}/files/rke2.linux-amd64.tar.gz"
```

**Пример: Удалённый URL**

```yaml
# group_vars/all.yml
---
rke2_install_tarball_url: "https://internal-repo.example.com/rke2/rke2.linux-amd64.tar.gz"
```

**Пример: Принудительный tarball с GitHub**

```yaml
# group_vars/all.yml
---
rke2_force_tarball_install: true
rke2_install_version: v1.29.12+rke2r1
```

### Образы контейнеров

Для изолированных сред предоставьте предзагруженные образы контейнеров:

| Переменная | Описание |
|------------|----------|
| `rke2_images_urls` | Список URL для скачивания образов |
| `rke2_images_local_tarball_path` | Список локальных архивов образов |

**Пример: Локальные образы**

```yaml
# group_vars/all.yml
---
rke2_images_local_tarball_path:
  - "{{ playbook_dir }}/files/rke2-images.linux-amd64.tar.zst"
  - "{{ playbook_dir }}/files/rke2-images-cilium.linux-amd64.tar.zst"
```

**Пример: Удалённые URL**

```yaml
# group_vars/all.yml
---
rke2_images_urls:
  - "https://internal-repo.example.com/rke2/rke2-images.linux-amd64.tar.zst"
```

---

## Полный пример

### Структура директорий

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
# Установка через tarball
rke2_install_local_tarball_path: "{{ playbook_dir }}/files/rke2.linux-amd64.tar.gz"

# Предзагруженные образы
rke2_images_local_tarball_path:
  - "{{ playbook_dir }}/files/rke2-images.linux-amd64.tar.zst"

# Конфигурация кластера
cluster_rke2_config:
  selinux: true
```

### Развёртывание

```bash
ansible-playbook site.yml -i inventory/hosts.yml -b
```

---

## Устранение неполадок

### Проблемы с загрузкой образов

Если образы не загружаются, проверьте:

1. Права доступа к файлам архивов образов
2. Достаточно места на диске в `/var/lib/rancher/rke2/agent/images/`
3. Правильный формат файлов (`.tar.zst` или `.tar.gz`)

### Ошибки SELinux

При проблемах, связанных с SELinux:

```bash
# Проверка запретов
ausearch -m avc -ts recent

# Установка политики SELinux
yum install -y rke2-selinux
```

### Проверка установки

```bash
# Проверка сервиса RKE2
systemctl status rke2-server

# Проверка загруженных образов
ls -la /var/lib/rancher/rke2/agent/images/

# Проверка containerd
/var/lib/rancher/rke2/bin/crictl images
```
