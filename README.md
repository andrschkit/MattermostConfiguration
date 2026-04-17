# MattermostConfiguration

`docker-compose.yaml` поднимает `Mattermost` и `PostgreSQL` для размещения чата за внешним `nginx`.

Параметры текущей схемы:

- внешний адрес чата: `https://chat.uaz-rosatom.ru`
- `nginx` уже проксирует трафик на `192.168.10.80:8065`
- сам `Mattermost` слушает HTTP внутри ВМ на порту `8065`

## Запуск

1. Скопировать `.env.example` в `.env`
2. Указать пароль для БД, параметры `Calls` и `SMTP`
3. Запустить контейнеры:

```bash
docker compose up -d
```

Пример `.env`:

```env
POSTGRES_PASSWORD=change_me_to_strong_password
CALLS_UDP_PORT=8443
CALLS_TCP_PORT=8443
CALLS_ICE_HOST_OVERRIDE=PUBLIC_IP_OR_FQDN
CALLS_ICE_HOST_PORT_OVERRIDE=8443
CALLS_DEFAULT_ENABLED=true
CALLS_ALLOW_ENABLE_CALLS=true
SMTP_AUTH_ENABLED=true
SMTP_SECURITY=STARTTLS
SMTP_SERVER=smtp.company.local
SMTP_PORT=587
SMTP_USERNAME=mattermost@uaz-rosatom.ru
SMTP_PASSWORD=change_me_to_smtp_password
SMTP_FROM=mattermost@uaz-rosatom.ru
```

## Что поднимается

- `postgres:16` для хранения данных `Mattermost`
- `mattermost/mattermost-team-edition:latest`
- постоянные Docker volumes для БД, конфигурации, данных, логов и плагинов
- публикация `8443/udp` и `8443/tcp` для встроенного `Mattermost Calls`

## Приглашения по email

После заполнения `SMTP`-параметров и перезапуска `Mattermost` можно приглашать пользователей через веб-интерфейс:

1. Войти под администратором
2. Открыть нужную команду
3. Выбрать `Invite People` или `Invite Members`
4. Ввести один или несколько `email`
5. Отправить приглашение

Если ваша почта не требует авторизации, можно указать:

```env
SMTP_AUTH_ENABLED=false
SMTP_USERNAME=
SMTP_PASSWORD=
```

## Важно для nginx

Так как TLS завершается на внешнем `nginx`, в `Mattermost` уже задан `SiteURL = https://chat.uaz-rosatom.ru`, а сам контейнер публикует `8065:8065`.

Для корректной работы веб-сокетов в `nginx` должны пробрасываться заголовки `Upgrade`, `Connection` и `X-Forwarded-Proto https`.

## Звонки

Для встроенного `Mattermost Calls` в `docker-compose.yaml` уже открыты:

- `8443/udp`
- `8443/tcp`

Дополнительная настройка сети и плагина описана в `CALLS_SETUP.md`.