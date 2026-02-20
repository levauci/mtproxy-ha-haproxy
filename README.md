# Active-Active MTProxy + HAProxy + TPROXY

Два MTProxy-инстанса на одной машине, TCP-балансировка через HAProxy с failover и сохранением реального IP клиента (TPROXY).

## Почему выбрано это решение

| Компонент | Зачем |
|-----------|-------|
| **HAProxy (TCP mode)** | L4-балансировка с health-check, graceful reload без разрыва соединений |
| **TPROXY + transparent** | Сохранение исходного IP клиента в пакетах (в отличие от DNAT/SNAT, backend видит реальный адрес). Альтернатива — PROXY protocol — не поддерживается MTProxy, а X-Forwarded-For неприменим для TCP |
| **leastconn** | Равномерное распределение при разном числе активных подключений на backend-ах |
| **CAP_NET_ADMIN / CAP_NET_RAW** | HAProxy работает от непривилегированного пользователя `haproxy`, но получает Linux capabilities через systemd drop-in для создания transparent-сокетов |
| **Ansible роли** | Разделение на `common`, `tproxy`, `mtproxy`, `haproxy` — каждая роль самодокументирована (`defaults/main.yml`, `meta/main.yml`) |

## Архитектура

```text
Client -> HAProxy (port 443, TCP, transparent)
          ├─> mtproxy1 (127.0.0.1:10000)
          └─> mtproxy2 (127.0.0.1:10001)
```

HAProxy принимает подключения на порт 443 с опцией `transparent`, сохраняя реальный IP клиента. Iptables TPROXY + policy routing обеспечивают корректную маршрутизацию пакетов. Health checks (`tcp-check connect`, `inter 2000 fall 3 rise 2`) автоматически выводят недоступные backend-ы из балансировки.

## Требования

- Vagrant + [vagrant-qemu](https://github.com/ppggff/vagrant-qemu) plugin
- Ansible >= 2.12

## Переменные

Единый файл переменных: `group_vars/all.yml`

| Переменная | По умолчанию | Описание |
|------------|-------------|----------|
| `mtproxy_replicas` | `2` | Количество MTProxy контейнеров |
| `mtproxy_image` | `telegrammessenger/proxy:latest` | Docker-образ |
| `mtproxy_secret` | `dd0000...` | Секрет для Telegram-клиентов (32 hex) |
| `mtproxy_base_port` | `10000` | Базовый порт (контейнеры получают 10000, 10001, …) |
| `haproxy_port` | `443` | Внешний порт приёма соединений |
| `haproxy_balance_algorithm` | `leastconn` | Алгоритм балансировки |

Каждая роль также содержит `defaults/main.yml` с дефолтами для своих переменных.

## Развёртывание

```bash
# Поднять VM и задеплоить (Ansible запускается автоматически через Vagrant provisioner)
vagrant up --provider=qemu
```

Повторный деплой (например, после изменения переменных):

```bash
vagrant provision
```

## Проверка после deploy

```bash
vagrant ssh

# Два контейнера запущены
docker ps --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"
# NAMES       STATUS          PORTS
# mtproxy1    Up X minutes    0.0.0.0:10000->443/tcp
# mtproxy2    Up X minutes    0.0.0.0:10001->443/tcp

# HAProxy backend-ы UP
echo "show stat" | socat stdio /run/haproxy/admin.sock | grep mtp | cut -d, -f1,2,18
# mtproxies,mtp1,UP
# mtproxies,mtp2,UP

# Или через stats-страницу (доступна с хоста на http://127.0.0.1:8404/stats)
curl -s http://127.0.0.1:8404/stats | grep -oE "mtp[0-9]+</td><td>[^<]+"
```

## Проверка failover

```bash
vagrant ssh

# 1. Убедиться, что оба backend-а UP
echo "show stat" | socat stdio /run/haproxy/admin.sock | grep mtp | cut -d, -f1,2,18
# mtproxies,mtp1,UP
# mtproxies,mtp2,UP

# 2. Остановить второй инстанс
docker stop mtproxy2

# 3. Подождать ~6 сек (inter 2000 * fall 3) и проверить статус
sleep 7
echo "show stat" | socat stdio /run/haproxy/admin.sock | grep mtp | cut -d, -f1,2,18
# mtproxies,mtp1,UP
# mtproxies,mtp2,DOWN

# 4. В логах HAProxy видно событие
journalctl -u haproxy --no-pager -n 20 | grep -E "DOWN|UP"
# ... Server mtproxies/mtp2 is DOWN ...

# 5. Новые подключения продолжают обслуживаться через mtp1

# 6. Вернуть обратно
docker start mtproxy2
# Через ~4 сек (inter 2000 * rise 2):
sleep 5
echo "show stat" | socat stdio /run/haproxy/admin.sock | grep mtp | cut -d, -f1,2,18
# mtproxies,mtp1,UP
# mtproxies,mtp2,UP
```

**Итог**: при `docker stop mtproxy2` HAProxy автоматически переключает весь трафик на `mtproxy1`. Клиенты Telegram продолжают работать без разрыва.

## Масштабирование (scale)

Изменить число реплик в `group_vars/all.yml`:

```yaml
mtproxy_replicas: 3
```

Применить:

```bash
vagrant provision
```

Ansible:
1. Создаст недостающие контейнеры (`mtproxy3`)
2. Удалит лишние при scale-down
3. Перегенерирует `haproxy.cfg` из шаблона
4. Выполнит `systemctl reload haproxy` (без разрыва существующих соединений)

## Теги (избирательный запуск)

```bash
# Только масштабирование контейнеров + обновление HAProxy конфига
ansible-playbook playbooks/deploy.yml --tags scale

# Только обновление HAProxy конфига
ansible-playbook playbooks/deploy.yml --tags config

# Только smoke-тесты (проверить здоровье стека)
ansible-playbook playbooks/deploy.yml --tags verify
```

| Тег | Роли |
|-----|------|
| `setup` | common, tproxy |
| `scale` | mtproxy, haproxy |
| `config` | haproxy |
| `verify` | verify (smoke-тесты) |

## Ручной запуск Ansible (без Vagrant provisioner)

```bash
# Сгенерировать SSH-конфиг из Vagrant
vagrant ssh-config > .vagrant/ssh-config

# Запустить playbook
ansible-playbook playbooks/deploy.yml \
  --ssh-extra-args="-F .vagrant/ssh-config"
```

## Telegram-ссылка

```text
tg://proxy?server=YOUR_SERVER_IP&port=8443&secret=dd00000000000000000000000000000000
```

(Порт `8443` — forwarded_port с хоста; на реальном сервере используйте `443` напрямую.)

## Структура ролей

```text
roles/
├── common/    — базовые пакеты (Docker, HAProxy, iptables, QEMU binfmt)
├── tproxy/    — sysctl, iptables TPROXY, systemd unit
├── mtproxy/   — Docker-контейнеры MTProxy (масштабируемые через mtproxy_replicas)
├── haproxy/   — конфигурация HAProxy, systemd drop-in для capabilities
└── verify/    — smoke-тесты (контейнеры, бэкенды UP, stats page)
```

Каждая роль содержит: `tasks/`, `handlers/`, `defaults/`, `meta/`, `templates/` (где применимо).

## Troubleshooting

| Проблема | Решение |
|----------|---------|
| `vagrant provision` — timeout SSH | `vagrant reload` или увеличить `config.ssh.connect_timeout` в `Vagrantfile` |
| HAProxy не стартует | `haproxy -c -f /etc/haproxy/haproxy.cfg` — проверить синтаксис конфига |
| TPROXY не работает | `iptables -t mangle -L -n` — убедиться что chain `HAPROXY_TPROXY` существует |
| Контейнер не стартует | `docker logs mtproxy1` — проверить секрет и образ |
| Нет доступа к stats socket | Убедиться что пользователь в группе `haproxy`: `groups vagrant` |
