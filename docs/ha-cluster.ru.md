# Конфигурация высокодоступного кластера

Это руководство описывает развертывание высокодоступных RKE2 кластеров с несколькими узлами управления.

## Содержание

- [Обзор](#обзор)
- [Требования](#требования)
- [Базовая HA конфигурация (3 сервера)](#базовая-ha-конфигурация-3-сервера)
- [HA с выделенным etcd](#ha-с-выделенным-etcd)
- [Мульти-зональное развертывание](#мульти-зональное-развертывание)
- [Конфигурация балансировщика нагрузки](#конфигурация-балансировщика-нагрузки)
- [Лучшие практики](#лучшие-практики)

## Обзор

Высокодоступный RKE2 кластер требует:
- **Минимум 3 серверных узла** для кворума etcd
- **Нечетное количество серверов** (3, 5, 7) для корректного выбора лидера
- **Балансировщик нагрузки** перед API серверами (рекомендуется)

## Требования

| Компонент | Минимум | Рекомендуется |
|-----------|---------|---------------|
| Серверные узлы | 3 | 3 или 5 |
| CPU на сервер | 2 ядра | 4 ядра |
| RAM на сервер | 4 ГБ | 8 ГБ |
| Диск на сервер | 50 ГБ SSD | 100 ГБ SSD |

## Базовая HA конфигурация (3 сервера)

### Инвентарь

```yaml
# inventory/hosts.yml
---
rke2_cluster:
  children:
    rke2_servers:
      hosts:
        server1.example.com:
          ansible_host: 10.0.0.10
        server2.example.com:
          ansible_host: 10.0.0.11
        server3.example.com:
          ansible_host: 10.0.0.12
    rke2_agents:
      hosts:
        worker1.example.com:
          ansible_host: 10.0.0.20
        worker2.example.com:
          ansible_host: 10.0.0.21
        worker3.example.com:
          ansible_host: 10.0.0.22
```

### Конфигурация кластера

```yaml
# inventory/group_vars/all.yml
---
# Используем первый сервер для начального bootstrap
rke2_kubernetes_api_server_host: "10.0.0.10"

# Или используем VIP балансировщика
# rke2_kubernetes_api_server_host: "10.0.0.100"

cluster_rke2_config:
  # Рекомендуется для HA кластеров
  cluster-cidr: "10.42.0.0/16"
  service-cidr: "10.43.0.0/16"
```

### Конфигурация серверов

```yaml
# inventory/group_vars/rke2_servers.yml
---
group_rke2_config:
  # Включаем снимки etcd для резервного копирования
  etcd-snapshot-schedule-cron: "0 */6 * * *"
  etcd-snapshot-retention: 5

  # TLS SANs для балансировщика
  tls-san:
    - "10.0.0.100"  # VIP балансировщика
    - "k8s.example.com"  # DNS имя
```

## HA с выделенным etcd

Для больших кластеров можно запускать etcd на выделенных узлах:

```yaml
# inventory/hosts.yml
---
rke2_cluster:
  children:
    rke2_servers:
      hosts:
        server1.example.com:
          ansible_host: 10.0.0.10
        server2.example.com:
          ansible_host: 10.0.0.11
        server3.example.com:
          ansible_host: 10.0.0.12
    rke2_agents:
      hosts:
        worker[1:10].example.com:
```

```yaml
# inventory/group_vars/rke2_servers.yml
---
group_rke2_config:
  # Запрет планирования рабочих нагрузок на control plane
  node-taint:
    - "CriticalAddonsOnly=true:NoExecute"
```

## Мульти-зональное развертывание

Распределите узлы по зонам доступности для отказоустойчивости:

```yaml
# inventory/hosts.yml
---
rke2_cluster:
  children:
    rke2_servers:
      hosts:
        server1-zone-a.example.com:
          ansible_host: 10.0.1.10
          zone: zone-a
        server2-zone-b.example.com:
          ansible_host: 10.0.2.10
          zone: zone-b
        server3-zone-c.example.com:
          ansible_host: 10.0.3.10
          zone: zone-c
    rke2_agents:
      children:
        zone_a_workers:
          hosts:
            worker1-zone-a.example.com:
              ansible_host: 10.0.1.20
        zone_b_workers:
          hosts:
            worker1-zone-b.example.com:
              ansible_host: 10.0.2.20
        zone_c_workers:
          hosts:
            worker1-zone-c.example.com:
              ansible_host: 10.0.3.20
```

```yaml
# inventory/host_vars/server1-zone-a.example.com.yml
---
host_rke2_config:
  node-label:
    - "topology.kubernetes.io/zone=zone-a"
```

## Конфигурация балансировщика нагрузки

### Пример HAProxy

```
# /etc/haproxy/haproxy.cfg
frontend k8s-api
    bind *:6443
    mode tcp
    option tcplog
    default_backend k8s-api-backend

backend k8s-api-backend
    mode tcp
    option tcp-check
    balance roundrobin
    server server1 10.0.0.10:6443 check fall 3 rise 2
    server server2 10.0.0.11:6443 check fall 3 rise 2
    server server3 10.0.0.12:6443 check fall 3 rise 2

frontend rke2-supervisor
    bind *:9345
    mode tcp
    option tcplog
    default_backend rke2-supervisor-backend

backend rke2-supervisor-backend
    mode tcp
    option tcp-check
    balance roundrobin
    server server1 10.0.0.10:9345 check fall 3 rise 2
    server server2 10.0.0.11:9345 check fall 3 rise 2
    server server3 10.0.0.12:9345 check fall 3 rise 2
```

### Пример Nginx

```nginx
# /etc/nginx/nginx.conf
stream {
    upstream k8s_api {
        least_conn;
        server 10.0.0.10:6443 max_fails=3 fail_timeout=30s;
        server 10.0.0.11:6443 max_fails=3 fail_timeout=30s;
        server 10.0.0.12:6443 max_fails=3 fail_timeout=30s;
    }

    upstream rke2_supervisor {
        least_conn;
        server 10.0.0.10:9345 max_fails=3 fail_timeout=30s;
        server 10.0.0.11:9345 max_fails=3 fail_timeout=30s;
        server 10.0.0.12:9345 max_fails=3 fail_timeout=30s;
    }

    server {
        listen 6443;
        proxy_pass k8s_api;
    }

    server {
        listen 9345;
        proxy_pass rke2_supervisor;
    }
}
```

### Использование балансировщика с Ansible

```yaml
# inventory/group_vars/all.yml
---
# Указываем на балансировщик вместо первого сервера
rke2_kubernetes_api_server_host: "lb.example.com"
```

```yaml
# inventory/group_vars/rke2_servers.yml
---
group_rke2_config:
  tls-san:
    - "lb.example.com"
    - "10.0.0.100"  # IP балансировщика
```

## Лучшие практики

### 1. Используйте нечетное количество серверов

Всегда используйте 3, 5 или 7 серверных узлов:
- **3 узла**: Переживает 1 отказ
- **5 узлов**: Переживает 2 отказа
- **7 узлов**: Переживает 3 отказа

### 2. Распределяйте по доменам отказа

Распределяйте серверные узлы по:
- Разным стойкам
- Разным зонам доступности
- Разным дата-центрам (с низкой задержкой)

### 3. Настройте снимки etcd

```yaml
group_rke2_config:
  etcd-snapshot-schedule-cron: "0 */4 * * *"  # Каждые 4 часа
  etcd-snapshot-retention: 10
  etcd-snapshot-dir: "/var/lib/rancher/rke2/server/db/snapshots"
```

### 4. Установите резервирование ресурсов

```yaml
group_rke2_config:
  kubelet-arg:
    - "system-reserved=cpu=500m,memory=500Mi"
    - "kube-reserved=cpu=500m,memory=500Mi"
```

### 5. Включите Pod Security

```yaml
cluster_rke2_config:
  profile: cis  # Или cis-1.23, cis-1.6
```

### 6. Используйте выделенный Control Plane

Для production, добавьте taint на узлы управления:

```yaml
# inventory/group_vars/rke2_servers.yml
group_rke2_config:
  node-taint:
    - "node-role.kubernetes.io/control-plane=true:NoSchedule"
```

### 7. Мониторьте здоровье etcd

Добавьте мониторинг для etcd:

```bash
# Проверка списка членов etcd
/var/lib/rancher/rke2/bin/etcdctl \
  --cacert=/var/lib/rancher/rke2/server/tls/etcd/server-ca.crt \
  --cert=/var/lib/rancher/rke2/server/tls/etcd/server-client.crt \
  --key=/var/lib/rancher/rke2/server/tls/etcd/server-client.key \
  member list

# Проверка здоровья endpoint'ов etcd
/var/lib/rancher/rke2/bin/etcdctl \
  --cacert=/var/lib/rancher/rke2/server/tls/etcd/server-ca.crt \
  --cert=/var/lib/rancher/rke2/server/tls/etcd/server-client.crt \
  --key=/var/lib/rancher/rke2/server/tls/etcd/server-client.key \
  endpoint health --cluster
```
