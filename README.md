# MTProxy HA — HAProxy + TPROXY

Два MTProxy-контейнера на одной машине, TCP-балансировка через HAProxy с failover и сохранением реального IP клиента (TPROXY).

## Архитектура

```
Client ──> HAProxy :443 (TCP, transparent)
             ├─> mtproxy1  127.0.0.1:10000
             └─> mtproxy2  127.0.0.1:10001
```

### Почему TPROXY, а не PROXY protocol / X-Forwarded-For

MTProxy не поддерживает PROXY protocol, а X-Forwarded-For неприменим для TCP.
TPROXY + `source 0.0.0.0 usesrc clientip` позволяет сохранить оригинальный IP клиента на уровне пакетов — backend видит реальный адрес без модификации протокола.

HAProxy работает в режиме `leastconn` — равномерное распределение при разном числе активных соединений.
Health checks (`tcp-check connect`, `inter 2000 fall 3 rise 2`) автоматически выводят упавшие backend-ы из балансировки.

## Требования

- Vagrant + [vagrant-qemu](https://github.com/ppggff/vagrant-qemu)
- Ansible >= 2.12

## Переменные

`group_vars/all.yml`:

| Переменная | По умолчанию | Описание |
|---|---|---|
| `mtproxy_replicas` | `2` | Количество MTProxy контейнеров |
| `mtproxy_secret` | `dd0000…` | Секрет для Telegram-клиентов (32 hex) |
| `mtproxy_base_port` | `10000` | Базовый порт (10000, 10001, …) |
| `haproxy_port` | `443` | Порт приёма соединений |
| `haproxy_balance_algorithm` | `leastconn` | Алгоритм балансировки |

## Развёртывание

```bash
vagrant up --provider=qemu
```

Повторный деплой:

```bash
vagrant provision
```

## Проверка failover

```bash
vagrant ssh

# убедиться, что оба backend-а UP
echo "show stat" | socat stdio /run/haproxy/admin.sock | grep mtp | cut -d, -f1,2,18

# остановить второй инстанс
docker stop mtproxy2

# подождать ~6 сек (inter 2000 × fall 3) и проверить
sleep 7
echo "show stat" | socat stdio /run/haproxy/admin.sock | grep mtp | cut -d, -f1,2,18
# mtproxies,mtp1,UP
# mtproxies,mtp2,DOWN

# новые подключения обслуживаются через mtp1

# вернуть обратно
docker start mtproxy2
sleep 5
echo "show stat" | socat stdio /run/haproxy/admin.sock | grep mtp | cut -d, -f1,2,18
# mtproxies,mtp1,UP
# mtproxies,mtp2,UP
```

При `docker stop mtproxy2` HAProxy переключает трафик на `mtproxy1`. Клиенты продолжают работать без разрыва.

## Масштабирование

Изменить `mtproxy_replicas` в `group_vars/all.yml` и применить:

```bash
vagrant provision
```

Ansible создаст/удалит контейнеры, перегенерирует `haproxy.cfg` и сделает `reload` без разрыва соединений.

## Теги

```bash
ansible-playbook playbooks/deploy.yml --tags scale    # контейнеры + haproxy конфиг
ansible-playbook playbooks/deploy.yml --tags config   # только haproxy конфиг
ansible-playbook playbooks/deploy.yml --tags verify   # smoke-тесты
```

## Структура ролей

```
roles/
├── common/    — Docker, HAProxy, iptables, базовые пакеты
├── tproxy/    — sysctl, iptables TPROXY, systemd unit
├── mtproxy/   — Docker-контейнеры MTProxy
├── haproxy/   — конфигурация HAProxy, systemd drop-in
└── verify/    — smoke-тесты
```
