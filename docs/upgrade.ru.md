# Руководство по обновлению RKE2 кластера

Это руководство описывает обновление RKE2 кластеров, развернутых с помощью этой Ansible коллекции.

## Содержание

- [Обзор](#обзор)
- [Чек-лист перед обновлением](#чек-лист-перед-обновлением)
- [Методы обновления](#методы-обновления)
- [Процедура rolling upgrade](#процедура-rolling-upgrade)
- [Процедура отката](#процедура-отката)
- [Совместимость версий](#совместимость-версий)

## Обзор

Обновления RKE2 должны выполняться аккуратно для минимизации простоя кластера. Общая стратегия:

1. **Резервное копирование** кластера перед любым обновлением
2. **Обновление серверов** по одному (control plane)
3. **Обновление агентов** (worker узлы)
4. **Проверка** здоровья кластера после каждого узла

## Чек-лист перед обновлением

Перед началом обновления:

- [ ] Изучить [release notes RKE2](https://github.com/rancher/rke2/releases) на предмет breaking changes
- [ ] Создать snapshot бэкап etcd
- [ ] Проверить текущее здоровье кластера
- [ ] Запланировать окно обслуживания
- [ ] Сначала протестировать обновление в staging среде
- [ ] Убедиться в достаточности ресурсов для rolling upgrade

### Создание бэкапа перед обновлением

```bash
# На любом серверном узле
/var/lib/rancher/rke2/bin/rke2 etcd-snapshot save \
  --name pre-upgrade-$(date +%Y%m%d-%H%M%S)
```

### Проверка здоровья кластера

```bash
# Проверка что все узлы Ready
kubectl get nodes

# Проверка системных подов
kubectl get pods -n kube-system

# Проверка здоровья etcd (на серверных узлах)
/var/lib/rancher/rke2/bin/etcdctl \
  --cacert=/var/lib/rancher/rke2/server/tls/etcd/server-ca.crt \
  --cert=/var/lib/rancher/rke2/server/tls/etcd/server-client.crt \
  --key=/var/lib/rancher/rke2/server/tls/etcd/server-client.key \
  endpoint health --cluster
```

## Методы обновления

### Метод 1: Используя Ansible (Рекомендуется)

Обновите версию в инвентаре:

```yaml
# inventory/group_vars/all.yml
rke2_install_version: "v1.30.0+rke2r1"  # Новая версия
```

Запустите плейбук с последовательным выполнением:

```bash
# Сначала обновите серверы (по одному)
ansible-playbook site.yml -i inventory/hosts.yml -b \
  --limit rke2_servers \
  -e "serial=1"

# Затем обновите агенты
ansible-playbook site.yml -i inventory/hosts.yml -b \
  --limit rke2_agents \
  -e "serial=1"
```

### Метод 2: Ручное обновление

Для большего контроля, обновляйте узлы вручную:

```bash
# На каждом узле остановите RKE2
systemctl stop rke2-server  # или rke2-agent

# Установите новую версию
curl -sfL https://get.rke2.io | INSTALL_RKE2_VERSION=v1.30.0+rke2r1 sh -

# Запустите сервис
systemctl start rke2-server  # или rke2-agent

# Проверьте что узел Ready
kubectl get nodes
```

### Метод 3: Обновление через каналы

Используйте каналы RKE2 для автоматического выбора версии:

```yaml
# inventory/group_vars/all.yml
rke2_channel: "stable"  # или "latest", "v1.30"
rke2_install_version: ""  # Пусто для использования канала
```

## Процедура Rolling Upgrade

### Шаг 1: Обновление первого сервера

```bash
# Вывод узла из планирования (опционально для HA кластеров)
kubectl drain server1 --ignore-daemonsets --delete-emptydir-data

# Обновление через Ansible
ansible-playbook site.yml -i inventory/hosts.yml -b --limit server1

# Возврат узла в планирование
kubectl uncordon server1

# Проверка что узел Ready и версия обновлена
kubectl get nodes -o wide
```

### Шаг 2: Обновление остальных серверов

Повторите для каждого серверного узла:

```bash
# Дождитесь полной готовности предыдущего узла
kubectl wait --for=condition=Ready node/server1 --timeout=300s

# Продолжите со следующим сервером
ansible-playbook site.yml -i inventory/hosts.yml -b --limit server2
```

### Шаг 3: Обновление агентов

```bash
# Обновление агентов (можно параллельно для больших кластеров)
ansible-playbook site.yml -i inventory/hosts.yml -b \
  --limit rke2_agents \
  -e "serial=3"  # 3 узла одновременно
```

### Шаг 4: Проверка после обновления

```bash
# Проверка что все узлы обновлены
kubectl get nodes -o wide

# Проверка системных подов
kubectl get pods -n kube-system

# Smoke тесты
kubectl run test --image=nginx --restart=Never
kubectl delete pod test
```

## Процедура отката

Если обновление не удалось, откатитесь к предыдущей версии:

### Вариант 1: Восстановление из снимка

```bash
# Остановка RKE2 на всех серверах
ansible -i inventory/hosts.yml rke2_servers -a "systemctl stop rke2-server" -b

# На первом сервере восстановите снимок
/var/lib/rancher/rke2/bin/rke2 server \
  --cluster-reset \
  --cluster-reset-restore-path=/var/lib/rancher/rke2/server/db/snapshots/pre-upgrade-xxx

# Запустите первый сервер
systemctl start rke2-server

# Присоедините остальные серверы (синхронизируются с восстановленным состоянием)
ansible-playbook site.yml -i inventory/hosts.yml -b --limit 'rke2_servers:!server1'
```

### Вариант 2: Переустановка предыдущей версии

```bash
# Обновите инвентарь с предыдущей версией
# inventory/group_vars/all.yml
rke2_install_version: "v1.29.0+rke2r1"  # Предыдущая версия

# Запустите плейбук
ansible-playbook site.yml -i inventory/hosts.yml -b
```

## Совместимость версий

### Поддерживаемые пути обновления

| С | На | Поддерживается |
|---|-----|----------------|
| v1.28.x | v1.29.x | Да |
| v1.29.x | v1.30.x | Да |
| v1.28.x | v1.30.x | Да (пропуск допустим) |

### Политика версий Kubernetes

- **kube-apiserver**: Может быть на 1 минорную версию впереди kubelet
- **kubelet**: Не может быть новее kube-apiserver
- **Рекомендуется**: Обновлять все узлы в одном окне обслуживания

### Совместимость компонентов

Всегда проверяйте совместимость с:
- CNI плагины (Canal, Calico, Cilium)
- Ingress контроллеры
- Storage драйверы
- Стек мониторинга

## Скрипт автоматизации обновления

Создайте скрипт для автоматического rolling upgrade:

```bash
#!/bin/bash
# upgrade-cluster.sh

set -e

NEW_VERSION="${1:-v1.30.0+rke2r1}"
INVENTORY="inventory/hosts.yml"

echo "=== Обновление RKE2 кластера до $NEW_VERSION ==="

# Бэкап перед обновлением
echo "Создание снимка перед обновлением..."
ansible -i $INVENTORY rke2_servers[0] -a \
  "/var/lib/rancher/rke2/bin/rke2 etcd-snapshot save --name pre-upgrade-$(date +%Y%m%d-%H%M%S)" -b

# Обновление версии в инвентаре
echo "Обновление версии в инвентаре..."
sed -i "s/rke2_install_version:.*/rke2_install_version: \"$NEW_VERSION\"/" inventory/group_vars/all.yml

# Обновление серверов
echo "Обновление серверных узлов..."
for server in $(ansible-inventory -i $INVENTORY --list | jq -r '.rke2_servers.hosts[]'); do
  echo "Обновление $server..."
  ansible-playbook site.yml -i $INVENTORY -b --limit $server

  echo "Ожидание готовности $server..."
  kubectl wait --for=condition=Ready node/$server --timeout=300s
done

# Обновление агентов
echo "Обновление агентов..."
ansible-playbook site.yml -i $INVENTORY -b --limit rke2_agents

# Проверка
echo "Проверка здоровья кластера..."
kubectl get nodes -o wide
kubectl get pods -n kube-system

echo "=== Обновление завершено ==="
```

## Устранение проблем при обновлении

### Узел завис в NotReady

```bash
# Проверка логов kubelet
journalctl -u rke2-server -f  # или rke2-agent

# Проверка ожидающих подов
kubectl get pods -A | grep -v Running
```

### Проблемы etcd после обновления

```bash
# Проверка логов etcd
journalctl -u rke2-server | grep etcd

# Проверка членов etcd
/var/lib/rancher/rke2/bin/etcdctl member list
```

### Откат одного узла

```bash
# Остановка RKE2
systemctl stop rke2-server

# Удаление текущей установки
/var/lib/rancher/rke2/bin/rke2-uninstall.sh

# Установка предыдущей версии
curl -sfL https://get.rke2.io | INSTALL_RKE2_VERSION=v1.29.0+rke2r1 sh -

# Запуск сервиса
systemctl start rke2-server
```
