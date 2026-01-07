# Руководство по резервному копированию и восстановлению

Это руководство описывает процедуры резервного копирования и восстановления для RKE2 кластеров, развернутых с помощью этой Ansible коллекции.

## Содержание

- [Обзор](#обзор)
- [Что резервировать](#что-резервировать)
- [Автоматические снимки etcd](#автоматические-снимки-etcd)
- [Ручное резервное копирование](#ручное-резервное-копирование)
- [Процедуры восстановления](#процедуры-восстановления)
- [Аварийное восстановление](#аварийное-восстановление)

## Обзор

RKE2 хранит все состояние кластера в etcd. Регулярное резервное копирование необходимо для аварийного восстановления. RKE2 предоставляет встроенную функциональность снимков etcd.

## Что резервировать

| Компонент | Расположение | Приоритет |
|-----------|--------------|-----------|
| Данные etcd | `/var/lib/rancher/rke2/server/db/` | Критичный |
| Снимки etcd | `/var/lib/rancher/rke2/server/db/snapshots/` | Критичный |
| Конфигурация RKE2 | `/etc/rancher/rke2/config.yaml` | Высокий |
| TLS сертификаты | `/var/lib/rancher/rke2/server/tls/` | Высокий |
| Токен | `/var/lib/rancher/rke2/server/node-token` | Высокий |
| Пользовательские манифесты | `/var/lib/rancher/rke2/server/manifests/` | Средний |

## Автоматические снимки etcd

### Включение автоматических снимков

Настройте автоматические снимки в инвентаре:

```yaml
# inventory/group_vars/rke2_servers.yml
---
group_rke2_config:
  # Создавать снимок каждые 6 часов
  etcd-snapshot-schedule-cron: "0 */6 * * *"

  # Хранить последние 10 снимков
  etcd-snapshot-retention: 10

  # Пользовательская директория снимков (опционально)
  etcd-snapshot-dir: "/var/lib/rancher/rke2/server/db/snapshots"

  # Префикс имени снимка (опционально)
  etcd-snapshot-name: "rke2-snapshot"
```

### Проверка автоматических снимков

```bash
# Список снимков
ls -la /var/lib/rancher/rke2/server/db/snapshots/

# Или используя команду rke2
/var/lib/rancher/rke2/bin/rke2 etcd-snapshot list
```

## Ручное резервное копирование

### Создание снимка по требованию

```bash
# На любом серверном узле
/var/lib/rancher/rke2/bin/rke2 etcd-snapshot save --name pre-upgrade-$(date +%Y%m%d-%H%M%S)
```

### Скрипт резервного копирования

Создайте комплексный скрипт резервного копирования:

```bash
#!/bin/bash
# /usr/local/bin/rke2-backup.sh

set -e

BACKUP_DIR="/backup/rke2/$(date +%Y%m%d-%H%M%S)"
SNAPSHOT_NAME="manual-backup-$(date +%Y%m%d-%H%M%S)"

# Создание директории для бэкапа
mkdir -p "$BACKUP_DIR"

echo "Создание снимка etcd..."
/var/lib/rancher/rke2/bin/rke2 etcd-snapshot save --name "$SNAPSHOT_NAME"

echo "Копирование снимка в директорию бэкапа..."
cp /var/lib/rancher/rke2/server/db/snapshots/${SNAPSHOT_NAME}* "$BACKUP_DIR/"

echo "Резервирование конфигурации..."
cp /etc/rancher/rke2/config.yaml "$BACKUP_DIR/" 2>/dev/null || true

echo "Резервирование токена..."
cp /var/lib/rancher/rke2/server/node-token "$BACKUP_DIR/" 2>/dev/null || true

echo "Резервирование TLS сертификатов..."
tar -czf "$BACKUP_DIR/tls-certs.tar.gz" -C /var/lib/rancher/rke2/server tls/ 2>/dev/null || true

echo "Создание архива..."
tar -czf "${BACKUP_DIR}.tar.gz" -C "$(dirname $BACKUP_DIR)" "$(basename $BACKUP_DIR)"
rm -rf "$BACKUP_DIR"

echo "Резервное копирование завершено: ${BACKUP_DIR}.tar.gz"
```

### Удаленное резервное копирование

Копирование бэкапов на удаленное хранилище:

```bash
# Используя rsync
rsync -avz /backup/rke2/ backup-server:/rke2-backups/

# Используя AWS S3
aws s3 sync /var/lib/rancher/rke2/server/db/snapshots/ s3://my-bucket/rke2-snapshots/

# Используя rclone
rclone sync /var/lib/rancher/rke2/server/db/snapshots/ remote:rke2-backups/
```

## Процедуры восстановления

### Восстановление из снимка (тот же кластер)

Используйте когда etcd поврежден, но сервер исправен:

```bash
# Остановка RKE2 на всех серверах
systemctl stop rke2-server

# На первом сервере восстановление из снимка
/var/lib/rancher/rke2/bin/rke2 server \
  --cluster-reset \
  --cluster-reset-restore-path=/var/lib/rancher/rke2/server/db/snapshots/<имя-снимка>

# Запуск RKE2
systemctl start rke2-server

# Проверка кластера
kubectl get nodes
```

### Восстановление на новый кластер

Используйте для аварийного восстановления на новую инфраструктуру:

1. **Подготовка нового сервера**

```bash
# Установка RKE2 (не запускать)
curl -sfL https://get.rke2.io | sh -
```

2. **Копирование снимка и конфигурации**

```bash
# Создание директорий
mkdir -p /var/lib/rancher/rke2/server/db/snapshots
mkdir -p /etc/rancher/rke2

# Копирование снимка
scp backup-server:/backups/snapshot-xxx.db /var/lib/rancher/rke2/server/db/snapshots/

# Копирование конфигурации (измените IP адреса при необходимости)
scp backup-server:/backups/config.yaml /etc/rancher/rke2/
```

3. **Восстановление кластера**

```bash
# Запуск с восстановлением
/var/lib/rancher/rke2/bin/rke2 server \
  --cluster-reset \
  --cluster-reset-restore-path=/var/lib/rancher/rke2/server/db/snapshots/<имя-снимка>
```

4. **Проверка и присоединение узлов**

```bash
# Проверка кластера
kubectl get nodes

# Присоединение остальных серверов и агентов через Ansible
ansible-playbook site.yml -i inventory/hosts.yml -b
```

### Восстановление одного члена etcd

Для HA кластеров когда один член etcd отказал:

```bash
# На отказавшем сервере остановка RKE2
systemctl stop rke2-server

# Удаление данных etcd
rm -rf /var/lib/rancher/rke2/server/db/etcd

# Запуск RKE2 (присоединится к кластеру)
systemctl start rke2-server

# Проверка членства etcd
/var/lib/rancher/rke2/bin/etcdctl \
  --cacert=/var/lib/rancher/rke2/server/tls/etcd/server-ca.crt \
  --cert=/var/lib/rancher/rke2/server/tls/etcd/server-client.crt \
  --key=/var/lib/rancher/rke2/server/tls/etcd/server-client.key \
  member list
```

## Аварийное восстановление

### Полная потеря кластера

Если все серверы потеряны:

1. **Подготовка новой инфраструктуры**
2. **Копирование последнего снимка на первый сервер**
3. **Восстановление используя процедуру для одного сервера**
4. **Присоединение дополнительных серверов**
5. **Проверка всех рабочих нагрузок**

### Чек-лист восстановления

- [ ] Последний бэкап доступен
- [ ] Целостность бэкапа проверена
- [ ] Новая инфраструктура подготовлена
- [ ] Сетевая связность подтверждена
- [ ] DNS записи обновлены (при необходимости)
- [ ] Снимок восстановлен
- [ ] Все узлы присоединены
- [ ] Системные поды здоровы
- [ ] Рабочие нагрузки запущены
- [ ] Ingress/LoadBalancer работает
- [ ] Persistent volumes доступны

### Тестирование бэкапов

Регулярно тестируйте процедуры восстановления:

```bash
# Создание тестового снимка
/var/lib/rancher/rke2/bin/rke2 etcd-snapshot save --name test-backup

# Проверка целостности снимка (на тестовой системе)
/var/lib/rancher/rke2/bin/rke2 server \
  --cluster-reset \
  --cluster-reset-restore-path=/path/to/test-backup
```

## Лучшие практики

1. **Частота бэкапов**: Минимум каждые 6 часов для production
2. **Ретеншн**: Хранить минимум 7 дней снимков
3. **Удаленное хранение**: Всегда копировать бэкапы на удаленное хранилище
4. **Тестирование восстановления**: Ежемесячные тесты восстановления
5. **Документация**: Поддерживать runbooks для типичных сценариев
6. **Мониторинг бэкапов**: Алерты при сбоях резервного копирования
7. **Шифрование**: Защита данных при передаче и хранении

## Мониторинг статуса бэкапов

Добавьте мониторинг для здоровья бэкапов:

```yaml
# Пример алерта Prometheus
groups:
  - name: rke2-backup
    rules:
      - alert: RKE2BackupMissing
        expr: time() - rke2_etcd_snapshot_last_success_timestamp > 86400
        for: 1h
        labels:
          severity: warning
        annotations:
          summary: "RKE2 etcd бэкап не создавался более 24 часов"
```
