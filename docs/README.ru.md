# Руководство по настройке RKE2 Ansible

> **Язык**: [English](./README.md) | [Русский](./README.ru.md)

## Содержание

- [Начало работы](#начало-работы)
  - [Методы установки](#методы-установки)
  - [Структура репозитория](#структура-репозитория)
- [Конфигурация инвентаря](#конфигурация-инвентаря)
  - [Минимальный инвентарь](#минимальный-инвентарь)
  - [Организация переменных](#организация-переменных)
- [Конфигурация RKE2](#конфигурация-rke2)
  - [Иерархия конфигурации](#иерархия-конфигурации)
  - [Указание версии RKE2](#указание-версии-rke2)
  - [Включение SELinux](#включение-selinux)
  - [CIS Hardening](#cis-hardening)
- [Расширенная конфигурация](#расширенная-конфигурация)
  - [Pod Security Admission](#pod-security-admission)
  - [Политика аудита](#политика-аудита)
  - [Пользовательские манифесты](#пользовательские-манифесты)
- [Доступные теги](#доступные-теги)
- [Справочник переменных](#справочник-переменных)
- [Примеры](#примеры)

---

## Начало работы

### Методы установки

#### Метод 1: Клонирование репозитория

Самый простой подход — клонировать и настроить:

```bash
git clone https://github.com/ashimov/rke2-ansible.git
cd rke2-ansible
cp -r docs/basic_sample_inventory inventory/my_cluster
```

#### Метод 2: Ansible Galaxy Collection

Добавьте в `requirements.yml`:

```yaml
collections:
  - name: ashimov.rke2_ansible
    source: https://github.com/ashimov/rke2-ansible.git
    type: git
    version: main
```

Установите и используйте:

```bash
ansible-galaxy collection install -r requirements.yml
```

```yaml
---
- name: Развёртывание RKE2
  hosts: all
  any_errors_fatal: true
  roles:
    - role: ashimov.rke2_ansible.rke2
```

### Структура репозитория

```
rke2-ansible/
├── roles/
│   ├── rke2/                    # Основная роль RKE2
│   │   ├── defaults/main.yml    # Переменные по умолчанию
│   │   ├── tasks/               # Файлы задач
│   │   └── handlers/            # Обработчики
│   └── testing/                 # Роль тестирования
├── docs/
│   ├── basic_sample_inventory/  # Простой пример инвентаря
│   └── advanced_sample_inventory/ # Полный пример
├── site.yml                     # Основной плейбук
└── testing.yml                  # Плейбук тестирования
```

---

## Конфигурация инвентаря

### Минимальный инвентарь

Простейший рабочий инвентарь:

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

Это развернёт последнюю стабильную версию RKE2 с настройками по умолчанию.

### Организация переменных

Для сложных развёртываний используйте `group_vars`:

```
inventory/
├── hosts.yml
└── group_vars/
    ├── all.yml           # Настройки всего кластера
    ├── rke2_servers.yml  # Настройки серверов
    └── rke2_agents.yml   # Настройки агентов
```

**Пример структуры для нескольких кластеров:**

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

## Конфигурация RKE2

### Иерархия конфигурации

Конфигурация RKE2 объединяется из трёх уровней (нижний переопределяет верхний):

| Уровень | Переменная | Область |
|---------|------------|---------|
| 1 | `cluster_rke2_config` | Все ноды |
| 2 | `group_rke2_config` | Группа серверов или агентов |
| 3 | `host_rke2_config` | Отдельный хост |

**Пример:**

```yaml
# group_vars/all.yml - применяется ко всем нодам
cluster_rke2_config:
  selinux: true
  write-kubeconfig-mode: "0640"

# group_vars/rke2_servers.yml - только серверы
group_rke2_config:
  profile: cis
  tls-san:
    - "kubernetes.example.com"

# hosts.yml - конкретный хост
rke2_servers:
  hosts:
    server1:
      host_rke2_config:
        node-label:
          - "topology.kubernetes.io/zone=us-east-1a"
```

### Указание версии RKE2

```yaml
# group_vars/all.yml
---
rke2_install_version: v1.29.12+rke2r1
```

Доступные версии на странице [RKE2 Releases](https://github.com/rancher/rke2/releases).

### Включение SELinux

```yaml
# group_vars/all.yml
---
cluster_rke2_config:
  selinux: true
```

Подробности в [документации RKE2 SELinux](https://docs.rke2.io/security/selinux).

### CIS Hardening

```yaml
# group_vars/rke2_servers.yml
---
group_rke2_config:
  profile: cis
```

Доступные профили: `cis`, `cis-1.23`

Подробности в [руководстве по усилению защиты RKE2](https://docs.rke2.io/security/hardening_guide).

---

## Расширенная конфигурация

### Pod Security Admission

**Шаг 1:** Создайте файл конфигурации PSA:

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

**Шаг 2:** Настройте роль:

```yaml
# group_vars/rke2_servers.yml
---
rke2_pod_security_admission_config_file_path: "{{ playbook_dir }}/files/pod-security-admission-config.yaml"

group_rke2_config:
  pod-security-admission-config-file: /etc/rancher/rke2/pod-security-admission-config.yaml
```

### Политика аудита

**Шаг 1:** Создайте политику аудита:

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

**Шаг 2:** Настройте роль:

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

### Пользовательские манифесты

Автоматическое развёртывание манифестов с помощью ресурсов HelmChart/HelmChartConfig.

**Манифесты до запуска** (перед стартом RKE2):

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

**Манифесты после запуска** (после старта RKE2):

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

## Доступные теги

Управляйте выполнением через `--tags` или `--skip-tags`:

| Тег | Описание |
|-----|----------|
| `always` | Выполняется всегда |
| `validation` | Предварительная проверка инвентаря |
| `facts` | Сбор фактов системы |
| `prereqs` | Предварительные требования ОС |
| `install` | Задачи установки |
| `tarball` | Установка через tarball |
| `rpm` | Установка через RPM |
| `airgap` | Обработка air-gap образов |
| `config` | Задачи конфигурации |
| `hardening` | CIS hardening |
| `bootstrap` | Начальная загрузка первого сервера |
| `join` | Операции присоединения нод |
| `token` | Управление токенами |
| `post-install` | Пост-установочные задачи |
| `utilities` | Настройка CLI инструментов |
| `manifests` | Развёртывание манифестов |

**Примеры:**

```bash
# Только установка, без конфигурации
ansible-playbook site.yml -i inventory/ --tags install

# Пропустить hardening
ansible-playbook site.yml -i inventory/ --skip-tags hardening
```

---

## Справочник переменных

### Установка

| Переменная | По умолчанию | Описание |
|------------|--------------|----------|
| `rke2_channel` | `stable` | Канал релизов |
| `rke2_install_version` | `""` | Конкретная версия (например, `v1.29.12+rke2r1`) |
| `rke2_force_tarball_install` | `false` | Принудительная установка через tarball |

### Пути

| Переменная | По умолчанию | Описание |
|------------|--------------|----------|
| `rke2_config_dir` | `/etc/rancher/rke2` | Директория конфигурации |
| `rke2_data_dir` | `/var/lib/rancher/rke2` | Директория данных |
| `rke2_bin_path` | `/var/lib/rancher/rke2/bin` | Путь к бинарникам |

### Сеть

| Переменная | По умолчанию | Описание |
|------------|--------------|----------|
| `rke2_api_port` | `6443` | Порт Kubernetes API |
| `rke2_supervisor_port` | `9345` | Порт Supervisor API |
| `rke2_add_iptables_rules` | `false` | Автодобавление правил iptables |

### Таймауты

| Переменная | По умолчанию | Описание |
|------------|--------------|----------|
| `rke2_api_wait_timeout` | `300` | Таймаут ожидания API сервера (сек) |
| `rke2_node_ready_retries` | `30` | Количество попыток проверки готовности ноды |
| `rke2_node_ready_delay` | `10` | Задержка между попытками (сек) |

---

## Примеры

Смотрите примеры инвентарей в этой директории:

- **[basic_sample_inventory](./basic_sample_inventory/)** — Минимальная конфигурация
- **[advanced_sample_inventory](./advanced_sample_inventory/)** — Полная конфигурация с CIS, PSA, манифестами
- **[tarball_sample_inventory](./tarball_sample_inventory/)** — Air-gap установка
