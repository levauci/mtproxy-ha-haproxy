# Active-Active MTProxy + HAProxy + TPROXY

Проект поднимает два MTProxy-инстанса на одной машине, балансирует TCP-подключения через HAProxy и поддерживает failover при остановке одного контейнера.

## Почему выбрано это решение

- **HAProxy в TCP-режиме**: надежный L4-балансировщик с health-check и reload без остановки процесса.
- **TPROXY + transparent bind**: позволяет сохранять исходный IP клиента в сетевых пакетах при проксировании.
- **Ansible роли**: конфигурация разбита на `common`, `tproxy`, `mtproxy`, `haproxy` для прозрачной поддержки и масштабирования.

## Архитектура

```text
Client -> HAProxy (port 443, TCP, transparent)
          ├─> mtproxy1 (127.0.0.1:10000)
          └─> mtproxy2 (127.0.0.1:10001)
```

## Требования

- Vagrant (QEMU provider)
- Ansible

## Переменные

Основные переменные находятся в `inventory/group_vars/all.yml`.

```yaml
mtproxy_replicas: 2
mtproxy_image: "telegrammessenger/proxy:latest"
mtproxy_secret: "dd00000000000000000000000000000000"
mtproxy_base_port: 10000
haproxy_port: 443
haproxy_stats_port: 8404
haproxy_balance_algorithm: "leastconn"
```

## Развёртывание

```bash
vagrant up --provider=qemu
ansible-playbook playbooks/deploy.yml
```

## Проверка после deploy

```bash
vagrant ssh

# 1) Проверить, что подняты два MTProxy контейнера
docker ps --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"

# 2) Проверить HAProxy backend-ы
curl -s http://127.0.0.1:8404/stats | grep -E "mtp1|mtp2"

# 3) Проверить health-check события
journalctl -u haproxy -n 100 --no-pager
```

## Проверка failover (кейс из задания)

```bash
# Остановить второй инстанс
docker stop mtproxy2

# Убедиться, что mtproxy2 стал DOWN, mtproxy1 остался UP
curl -s http://127.0.0.1:8404/stats | grep -E "mtp1|mtp2"

# Проверить, что HAProxy зафиксировал недоступность backend-а
journalctl -u haproxy -n 100 --no-pager | grep -E "mtp2|DOWN|UP"
```

Ожидаемый результат: HAProxy помечает `mtp2` как `DOWN`, продолжает проксировать трафик на `mtp1`, подключения продолжают обслуживаться.

## Масштабирование

Чтобы увеличить число MTProxy-инстансов, измените:

```yaml
mtproxy_replicas: 3
```

и повторно выполните:

```bash
ansible-playbook playbooks/deploy.yml
```

HAProxy конфиг генерируется из шаблона и применяется через reload (best effort, без полного простоя сервиса).

## Telegram ссылка

```text
tg://proxy?server=YOUR_SERVER_IP&port=443&secret=dd00000000000000000000000000000000
```
