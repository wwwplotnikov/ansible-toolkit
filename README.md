# ansible-toolkit

Роли и плейбуки Ansible для настройки серверов с нуля.
Целевая ОС: Ubuntu 26.04 LTS. Роли самостоятельные, их можно использовать по отдельности.

## Сценарий: веб-сервер в Docker

`playbooks/web-server.yml` берёт свежесозданный сервер и приводит его в состояние,
в котором снаружи доступны ровно два порта: SSH только по ключу и HTTP на 80,
который отдаёт nginx в контейнере.

48 задач, `changed=0` на повторном прогоне.

## Роли

| Роль | Назначение |
|---|---|
| `baseline` | Hostname, таймзона, базовые пакеты, автообновления безопасности |
| `users` | Админ-пользователь, SSH-ключ, sudo с валидацией через `visudo` |
| `ssh_hardening` | Вход только по ключу, root запрещён, белый список пользователей |
| `firewall` | ufw с default deny, лимит на SSH, фильтрация портов контейнеров |
| `docker` | Официальный репозиторий, конфиг демона, ротация логов |
| `web` | nginx через Compose, healthcheck, статика |

## Использование

    cp inventory/hosts.yml.example inventory/hosts.yml
    # указать ansible_host и ansible_private_key_file

    ansible-galaxy collection install -r requirements.yml

    # первый прогон, админа ещё нет
    ansible-playbook playbooks/web-server.yml -e connect_user=root

    # последующие прогоны идут от админа
    ansible-playbook playbooks/web-server.yml

Отдельные роли запускаются по тегам:

    ansible-playbook playbooks/web-server.yml --tags firewall

## Переменные

Основные переменные собраны в `inventory/group_vars/all.yml`.
Значения по умолчанию для каждой роли лежат в её `defaults/main.yml`.

| Переменная | По умолчанию | Назначение |
|---|---|---|
| `admin_user` | `deploy` | Имя создаваемого админа |
| `admin_ssh_key_file` | — | Путь к публичному ключу на управляющей машине |
| `ssh_hardening_enabled` | `true` | Предохранитель, при `false` sshd не трогается |
| `firewall_docker_allowed_tcp_ports` | `[80]` | Порты контейнеров, открытые наружу |
| `web_image` | `nginx:1.29-alpine` | Образ веб-сервера |
| `web_port` | `80` | Опубликованный порт |

## Безопасность

Конфиги, способные оборвать доступ к серверу, пишутся только после валидации:
`visudo -cf` для sudoers, `sshd -t` для sshd, разбор JSON для конфига Docker.

Роль `ssh_hardening` перед изменением конфига проверяет, что вход по ключу
для админа действительно работает, и останавливается, если нет.

Опубликованные порты контейнеров фильтруются в цепочке `DOCKER-USER`

## Требования

- Ansible 2.16 или новее
- Коллекции из `requirements.yml`
- Доступ к серверу по SSH с правами root на первом прогоне
