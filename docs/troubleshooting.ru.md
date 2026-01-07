# Руководство по устранению неполадок

Это руководство охватывает типичные проблемы и их решения при развертывании RKE2 кластеров с помощью этой Ansible коллекции.

## Содержание

- [Ошибки присоединения узлов](#ошибки-присоединения-узлов)
- [Проблемы с API сервером](#проблемы-с-api-сервером)
- [Проблемы с токеном](#проблемы-с-токеном)
- [Проблемы CIS Hardening](#проблемы-cis-hardening)
- [Сетевые проблемы](#сетевые-проблемы)
- [Проблемы с сертификатами](#проблемы-с-сертификатами)
- [Сбор диагностической информации](#сбор-диагностической-информации)

## Ошибки присоединения узлов

### Симптом: Узел не может присоединиться к кластеру

**Возможные причины:**

1. **Несоответствие или истечение токена**
   ```bash
   # Проверка токена на серверном узле
   cat /var/lib/rancher/rke2/server/node-token

   # Проверка токена в конфигурации агента
   cat /etc/rancher/rke2/config.yaml
   ```

2. **Проблемы сетевого подключения**
   ```bash
   # Тест подключения к API серверу с агента
   curl -k https://<server-ip>:9345/cacerts

   # Проверка правил firewall
   iptables -L -n | grep -E '(6443|9345)'
   ```

3. **Проблемы разрешения DNS**
   ```bash
   # Проверка разрешения имени хоста
   getent hosts <server-hostname>
   ```

**Решение:**

Убедитесь, что переменная `rke2_kubernetes_api_server_host` указывает на доступный сервер:

```yaml
# inventory/group_vars/all.yml
rke2_kubernetes_api_server_host: "10.0.0.10"  # Используйте IP если DNS ненадежен
```

### Симптом: Таймаут ожидания готовности узла

**Возможные причины:**

1. Медленная сеть или дисковый I/O
2. Недостаточно ресурсов (CPU/RAM)
3. Образы контейнеров еще загружаются

**Решение:**

Увеличьте значения таймаутов:

```yaml
# inventory/group_vars/all.yml
rke2_api_wait_timeout: 600
rke2_node_ready_timeout: 600
rke2_node_ready_retries: 60
rke2_node_ready_delay: 15
```

## Проблемы с API сервером

### Симптом: API сервер не отвечает

**Проверка статуса сервиса:**

```bash
systemctl status rke2-server
journalctl -u rke2-server -f
```

**Проверка прослушивания порта:**

```bash
ss -tlnp | grep 6443
```

**Типичные решения:**

1. Перезапуск сервиса:
   ```bash
   systemctl restart rke2-server
   ```

2. Проверка здоровья etcd:
   ```bash
   /var/lib/rancher/rke2/bin/etcdctl \
     --cacert=/var/lib/rancher/rke2/server/tls/etcd/server-ca.crt \
     --cert=/var/lib/rancher/rke2/server/tls/etcd/server-client.crt \
     --key=/var/lib/rancher/rke2/server/tls/etcd/server-client.key \
     endpoint health
   ```

## Проблемы с токеном

### Симптом: Файл токена не найден

Файл токена создается после запуска первого сервера. Если он не существует:

```bash
# Проверка работы RKE2 сервера
systemctl status rke2-server

# Токен должен быть здесь
ls -la /var/lib/rancher/rke2/server/node-token
```

### Симптом: Таймаут получения токена

Увеличьте таймаут ожидания API:

```yaml
rke2_api_wait_timeout: 600
```

## Проблемы CIS Hardening

### Симптом: Плейбук падает во время CIS hardening

**Проверка существования конфигурации sysctl:**

```bash
# Для RPM установки
ls -la /usr/share/rke2/rke2-cis-sysctl.conf

# Для tarball установки
ls -la /usr/local/share/rke2/rke2-cis-sysctl.conf
```

### Симптом: Узел требует перезагрузки после изменений CIS

Это ожидаемое поведение. Плейбук автоматически перезагрузит узел если:
- Включен CIS профиль
- Изменилась конфигурация sysctl
- RKE2 уже был запущен

Для настройки таймаута перезагрузки:

```yaml
rke2_reboot_timeout: 600
```

## Сетевые проблемы

### Симптом: Поды не могут общаться между узлами

**Проверка конфигурации CNI:**

```bash
# Проверка CNI плагинов
ls -la /var/lib/rancher/rke2/agent/etc/cni/net.d/

# Проверка статуса Flannel или Canal
kubectl get pods -n kube-system | grep -E '(flannel|canal)'
```

**Проверка правил iptables:**

```bash
iptables -L -n -t nat
iptables -L -n -t filter
```

### Симптом: Service discovery не работает

```bash
# Проверка статуса CoreDNS
kubectl get pods -n kube-system -l k8s-app=kube-dns

# Тест DNS разрешения
kubectl run -it --rm debug --image=busybox -- nslookup kubernetes.default
```

## Проблемы с сертификатами

### Симптом: Сертификат истек или недействителен

**Проверка срока действия сертификата:**

```bash
openssl x509 -in /var/lib/rancher/rke2/server/tls/server-ca.crt -noout -dates
```

**Ротация сертификатов:**

```bash
# Остановка RKE2
systemctl stop rke2-server

# Удаление старых сертификатов
rm -rf /var/lib/rancher/rke2/server/tls

# Запуск RKE2 (перегенерирует сертификаты)
systemctl start rke2-server
```

## Сбор диагностической информации

### Базовый сбор информации

```bash
# Информация о системе
uname -a
cat /etc/os-release

# Версия RKE2
/var/lib/rancher/rke2/bin/rke2 --version

# Статус сервисов
systemctl status rke2-server rke2-agent

# Последние логи
journalctl -u rke2-server --since "1 hour ago"
journalctl -u rke2-agent --since "1 hour ago"
```

### Состояние Kubernetes кластера

```bash
# Статус узлов
kubectl get nodes -o wide

# Статус всех подов
kubectl get pods -A

# События
kubectl get events -A --sort-by='.lastTimestamp'
```

### Сетевая диагностика

```bash
# Проверка прослушиваемых портов
ss -tlnp

# Проверка iptables
iptables-save > /tmp/iptables-rules.txt

# Проверка маршрутов
ip route show
```

### Создание диагностического архива

```bash
#!/bin/bash
BUNDLE_DIR="/tmp/rke2-diagnostics-$(date +%Y%m%d-%H%M%S)"
mkdir -p "$BUNDLE_DIR"

# Информация о системе
uname -a > "$BUNDLE_DIR/system-info.txt"
cat /etc/os-release >> "$BUNDLE_DIR/system-info.txt"

# Конфигурация RKE2
cp /etc/rancher/rke2/config.yaml "$BUNDLE_DIR/" 2>/dev/null

# Логи сервисов
journalctl -u rke2-server --since "2 hours ago" > "$BUNDLE_DIR/rke2-server.log" 2>/dev/null
journalctl -u rke2-agent --since "2 hours ago" > "$BUNDLE_DIR/rke2-agent.log" 2>/dev/null

# Состояние Kubernetes
kubectl get nodes -o wide > "$BUNDLE_DIR/nodes.txt" 2>/dev/null
kubectl get pods -A > "$BUNDLE_DIR/pods.txt" 2>/dev/null
kubectl get events -A --sort-by='.lastTimestamp' > "$BUNDLE_DIR/events.txt" 2>/dev/null

# Создание архива
tar -czf "$BUNDLE_DIR.tar.gz" -C /tmp "$(basename $BUNDLE_DIR)"
echo "Диагностический архив создан: $BUNDLE_DIR.tar.gz"
```

## fapolicyd на RHEL 8+

Если вы используете fapolicyd, добавьте эти правила перед установкой:

```bash
cat <<-EOF >> /etc/fapolicyd/rules.d/80-rke2.rules
allow perm=any all : dir=/var/lib/rancher/
allow perm=any all : dir=/opt/cni/
allow perm=any all : dir=/run/k3s/
allow perm=any all : dir=/var/lib/kubelet/
EOF

systemctl restart fapolicyd
```

## Получение помощи

Если проблемы продолжаются:

1. Проверьте [документацию RKE2](https://docs.rke2.io/)
2. Поищите в [GitHub Issues](https://github.com/ashimov/rke2-ansible/issues)
3. Откройте новый issue с вашим диагностическим архивом
