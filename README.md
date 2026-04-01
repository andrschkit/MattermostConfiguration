# MattermostConfiguration

`docker-compose.yaml` поднимает `Mattermost` и `PostgreSQL` для размещения чата за внешним `nginx`.

Параметры текущей схемы:

- внешний адрес чата: ``
- `nginx` уже проксирует трафик на `192.168._._:8065`
- сам `Mattermost` слушает HTTP внутри ВМ на порту `8065`

## Запуск

1. Создать рядом с `docker-compose.yaml` файл `.env`
2. Указать пароль для БД:

```env
POSTGRES_PASSWORD=change_me_to_strong_password
```

3. Запустить контейнеры:

```bash
docker compose up -d
```

## Что поднимается

- `postgres:16` для хранения данных `Mattermost`
- `mattermost/mattermost-team-edition:latest`
- постоянные Docker volumes для БД, конфигурации, данных, логов и плагинов

## Важно для nginx

Так как TLS завершается на внешнем `nginx`, в `Mattermost` уже задан `SiteURL = `, а сам контейнер публикует `8065:8065`.

Для корректной работы веб-сокетов в `nginx` должны пробрасываться заголовки `Upgrade` и `Connection`, а также `X-Forwarded-Proto https`.