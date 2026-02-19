# Active-Active MTProxy + HAProxy Transparent

High-availability MTProxy setup with HAProxy load balancing and TPROXY for transparent proxying.

## Архитектура

```
Client -> HAProxy (443, TPROXY) -> MTProxy-0 (10000)
                                -> MTProxy-1 (10001)
```

- **HAProxy** принимает трафик на порту 443 с сохранением реального IP клиента (TPROXY)
- **MTProxy** контейнеры работают в режиме active-active с round-robin балансировкой

## Требования

- Vagrant + QEMU provider (для macOS ARM)
- Ansible

## Как развернуть

```bash
# Поднять VM
vagrant up --provider=qemu

# Задеплоить MTProxy + HAProxy
ansible-playbook playbooks/deploy.yml
```

## Конфигурация

Редактируй `group_vars/all.yml`:

```yaml
mtproxy_replicas: 2                              # количество MTProxy контейнеров
mtproxy_image: "telegrammessenger/proxy:latest"
mtproxy_secret: "dd000000..."                    # твой секрет (32 hex символа)
mtproxy_base_port: 10000                         # базовый порт для контейнеров
haproxy_port: 443                                # внешний порт
```

## Проверка

```bash
# SSH в VM
vagrant ssh

# Статус контейнеров
docker ps

# Статистика HAProxy
curl http://localhost:8404/stats

# Логи HAProxy
journalctl -u haproxy -f
```

## Ссылка для Telegram

```
tg://proxy?server=YOUR_SERVER_IP&port=443&secret=dd00000000000000000000000000000000
```
## Как развернуть
```bash
vagrant up --provider=qemu
ansible-playbook playbooks/deploy.yml
