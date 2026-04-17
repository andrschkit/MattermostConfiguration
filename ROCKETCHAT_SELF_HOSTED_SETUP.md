# Rocket.Chat Self-Hosted для команды до 50 человек

Этот документ описывает практическое решение для self-hosted `Rocket.Chat` под ваш сценарий:

- команда до `50` человек
- публичные и приватные каналы
- групповые звонки
- `SSO`
- интеграция с `GitLab`
- развёртывание в `Docker`

Сразу важный вывод: если требование по `PostgreSQL` строгое, то `Rocket.Chat` вам не подходит как основная чат-платформа, потому что для работы ему нужен `MongoDB` и рекомендуемый формат для production - `MongoDB Replica Set`. `PostgreSQL` как основная БД для Rocket.Chat не поддерживается.

Если `PostgreSQL` лишь желателен, а не обязателен, то рекомендуемая схема для Rocket.Chat выглядит так:

- `Rocket.Chat` для чатов и каналов
- `MongoDB` для хранения данных Rocket.Chat
- `Jitsi` для групповых аудио/видеозвонков
- `GitLab OAuth/OIDC` для входа пользователей
- `GitLab Webhook` для уведомлений из репозиториев, merge request, pipeline и issue

## Подходит ли бесплатная self-hosted версия под ваш кейс

Для вашего сценария бесплатной self-hosted установки Rocket.Chat обычно достаточно.

Что закрывает ваш кейс:

- каналы: публичные и приватные
- direct messages и групповые комнаты
- треды, поиск, файлы, упоминания, роли
- self-hosted развёртывание
- `OAuth/OIDC`-авторизация, в том числе через внешний провайдер вроде `GitLab`
- интеграции с `GitLab` через webhook
- групповые звонки через приложение `Jitsi`

Что стоит учитывать:

- звонки в Rocket.Chat обычно строятся не как собственный media stack Rocket.Chat, а через интеграцию с `Jitsi`
- часть расширенных возможностей звонков относится к платным возможностям, например запись звонков, история звонков и некоторые расширенные enterprise-функции
- для стабильных видеозвонков при росте нагрузки лучше держать `Jitsi` отдельно от Rocket.Chat

Итог: для команды до `50` человек решение подходит, если вы готовы использовать `MongoDB`, а для звонков - `Jitsi`.

## Рекомендуемая архитектура

Для вашего размера команды рекомендую не пытаться делать всё в одном контейнере и не привязывать звонки к тому же узлу, где база и чат.

Практичная схема:

1. `chat.example.com` -> `Rocket.Chat + MongoDB + Traefik` на одном сервере
2. `meet.example.com` -> `Jitsi` на отдельном сервере или хотя бы отдельном Docker host
3. `gitlab.example.com` -> ваш `GitLab`, который используется:
   - как `SSO`-провайдер
   - как источник webhook-событий

Почему так лучше:

- проще обновлять `Rocket.Chat` и `Jitsi` независимо
- меньше риск, что видеозвонки начнут душить чат и БД
- проще масштабировать звонки отдельно

## Почему не PostgreSQL

Это принципиальное ограничение платформы:

- `Rocket.Chat` хранит основные данные в `MongoDB`
- production-развёртывание опирается на `MongoDB Replica Set`
- официальная `Docker Compose`-схема Rocket.Chat также строится вокруг `MongoDB`

Если вам нужен именно стек на `PostgreSQL`, то лучше выбрать:

- `Mattermost`, если нужен похожий корпоративный чат
- либо другой мессенджер, который официально поддерживает `PostgreSQL`

Но если цель именно `Rocket.Chat`, то ниже приведено корректное и поддерживаемое решение на `MongoDB`.

## Целевая схема для production

### Хост 1: Rocket.Chat

Сервисы:

- `Rocket.Chat`
- `MongoDB`
- `NATS` (идёт в официальной compose-схеме Rocket.Chat)
- `Traefik`

Публичный адрес:

- `https://chat.example.com`

Открытые порты:

- `80/tcp`
- `443/tcp`

### Хост 2: Jitsi

Сервисы:

- `web`
- `prosody`
- `jicofo`
- `jvb`

Публичный адрес:

- `https://meet.example.com`

Открытые порты:

- `80/tcp`
- `443/tcp`
- `10000/udp`

## Минимальные рекомендации по ресурсам

Для старта под команду до `50` человек:

- сервер Rocket.Chat: `4 vCPU`, `8 GB RAM`, быстрый SSD
- сервер Jitsi: `4 vCPU`, `8 GB RAM` минимум, если планируются регулярные групповые звонки

Если ожидаются частые большие видеозвонки, Jitsi лучше усиливать отдельно, не увеличивая без необходимости ресурсы MongoDB и Rocket.Chat.

## DNS

До старта создайте DNS-записи:

- `chat.example.com` -> IP сервера Rocket.Chat
- `meet.example.com` -> IP сервера Jitsi

Если используете self-managed `GitLab`, то он уже должен быть доступен по своему домену, например:

- `gitlab.example.com`

## Развёртывание Rocket.Chat в Docker

Ниже используется официальный способ развёртывания Rocket.Chat через репозиторий `rocketchat-compose`.

### 1. Подготовка сервера

На Linux-сервере установите:

- `Docker`
- `Docker Compose v2`
- `Git`

### 2. Клонирование официального compose-репозитория

```bash
mkdir -p /opt/rocketchat
cd /opt/rocketchat
git clone --depth 1 https://github.com/RocketChat/rocketchat-compose.git .
cp .env.example .env
```

### 3. Настройка `.env`

Минимально рекомендую указать такие параметры:

```env
RELEASE=8.2.0
DOMAIN=chat.example.com
ROOT_URL=https://chat.example.com
LETSENCRYPT_ENABLED=true
LETSENCRYPT_EMAIL=it@example.com
TRAEFIK_PROTOCOL=https

# По желанию: мониторинг
GRAFANA_DOMAIN=
GRAFANA_PATH=
GRAFANA_ADMIN_PASSWORD=change_me_to_strong_password
```

Замечания:

- лучше фиксировать конкретную версию в `RELEASE`, а не использовать `latest`
- если мониторинг не нужен, можно не использовать соответствующий compose-файл при запуске
- `compose.database.yml` из официального стека поднимает `MongoDB` и `NATS`

### 4. Запуск Rocket.Chat

Вариант, максимально близкий к официальной документации:

```bash
cd /opt/rocketchat
docker compose -f compose.database.yml -f compose.monitoring.yml -f compose.traefik.yml -f compose.yml -f docker.yml up -d
```

Если мониторинг не нужен, можно упростить стек позже, но для первого запуска удобнее использовать официальный рекомендуемый путь.

### 5. Проверка запуска

```bash
docker compose ps
docker compose logs -f
```

После старта откройте:

- `https://chat.example.com`

И завершите initial setup wizard:

- создайте администратора
- задайте имя workspace
- проверьте вход и создание каналов

## Развёртывание Jitsi в Docker

Для групповых звонков в Rocket.Chat самым практичным вариантом будет `Jitsi`.

### 1. Подготовка сервера Jitsi

На отдельном Linux-сервере:

- установите `Docker`
- установите `Docker Compose v2`
- откройте `80/tcp`, `443/tcp`, `10000/udp`

### 2. Клонирование официального `docker-jitsi-meet`

```bash
mkdir -p /opt/jitsi
cd /opt/jitsi
git clone --depth 1 https://github.com/jitsi/docker-jitsi-meet.git .
cp env.example .env
./gen-passwords.sh
```

### 3. Базовая настройка `.env`

Минимальный набор:

```env
PUBLIC_URL=https://meet.example.com
ENABLE_LETSENCRYPT=1
LETSENCRYPT_DOMAIN=meet.example.com
LETSENCRYPT_EMAIL=it@example.com
HTTP_PORT=80
HTTPS_PORT=443
JVB_PORT=10000
TZ=Europe/Moscow
```

### 4. Запуск Jitsi

```bash
cd /opt/jitsi
docker compose up -d
```

### 5. Проверка

Проверьте, что открывается:

- `https://meet.example.com`

И что тестовая встреча создаётся без ошибок.

## Интеграция Jitsi в Rocket.Chat

После запуска Rocket.Chat и Jitsi:

1. В Rocket.Chat зайдите в `Marketplace`
2. Установите приложение `Jitsi`
3. Перейдите в настройки приложения
4. Укажите:
   - `Domain` = `meet.example.com`
   - `Use SSL` = `true`
5. Затем откройте `Manage > Workspace > Settings > Conference Call`
6. Выберите `Jitsi` как `Default Provider`
7. Включите звонки для:
   - `DM`
   - `Public channels`
   - `Private channels`
   - `Teams`, если используете их

После этого звонки можно будет запускать:

- кнопкой звонка в комнате
- командой `/jitsi`

## Настройка SSO через GitLab

Для вашей задачи самый удобный путь - использовать `GitLab` как `OAuth/OIDC`-провайдер.

### Вариант A: рекомендуемый

Если ваш `GitLab` поддерживает `OpenID Connect`, используйте его как identity provider.

Типовая discovery-точка:

- `https://gitlab.example.com/.well-known/openid-configuration`

Если в вашей версии Rocket.Chat удобнее настраивать `custom OAuth`, это тоже нормально. На практике для GitLab часто используют именно custom OAuth с OIDC scope.

### 1. Создайте приложение в GitLab

В `GitLab` создайте OAuth application.

Callback URL укажите такой:

```text
https://chat.example.com/_oauth/gitlab
```

Рекомендуемые scope:

```text
openid profile email
```

Сохраните:

- `Application ID`
- `Secret`

### 2. Создайте custom OAuth в Rocket.Chat

Откройте:

- `Manage > Workspace > Settings > OAuth`

Нажмите:

- `Add custom OAuth`

Имя метода:

- `gitlab`

Рекомендуемые поля:

- `URL`: `https://gitlab.example.com`
- `Authorize Path`: `/oauth/authorize`
- `Token Path`: `/oauth/token`
- `Identity Path`: `/oauth/userinfo`
- `Scope`: `openid profile email`
- `Id`: ваш `Application ID`
- `Secret`: ваш `Secret`
- `Button Text`: `Войти через GitLab`
- `Key Field`: `Email`
- `Email field`: `email`
- `Name field`: `name`
- `Username field`: `nickname`
- `Show Button on Login Page`: `true`

Рекомендую также включить по ситуации:

- `Merge users` = `true`, если у вас уже есть локальные пользователи с теми же email
- `Merge users from distinct services` = `true`, если хотите аккуратно сшить существующие учётки

### 3. Проверка

Проверьте:

1. выход из Rocket.Chat
2. появление кнопки `Войти через GitLab`
3. успешный логин тестового пользователя
4. корректное создание или привязку пользователя по email

## Интеграция Rocket.Chat с GitLab через webhook

Это нужно не для входа, а для событий из репозиториев:

- push
- merge request
- issue
- comments
- pipelines
- deployments

### 1. Создайте integration в Rocket.Chat

Откройте:

- `Manage > Workspace > Integrations`

Далее:

1. `New`
2. вкладка `Incoming`
3. заполните:
   - имя интеграции
   - канал, например `#gitlab`
   - от какого пользователя публиковать сообщения
4. включите `Scripts Enabled`
5. вставьте официальный script parser для `GitLab webhook` из документации Rocket.Chat

После сохранения Rocket.Chat покажет:

- `Webhook URL`
- `Secret Token`

### 2. Настройте webhook в GitLab

В проекте `GitLab`:

1. откройте `Settings > Webhooks`
2. вставьте `Webhook URL`
3. вставьте `Secret Token`
4. выберите нужные события:
   - `Push events`
   - `Merge request events`
   - `Issue events`
   - `Note events`
   - `Pipeline events`
5. сохраните webhook

### 3. Тест

Нажмите `Test` на стороне GitLab и убедитесь, что сообщение пришло в выбранный канал Rocket.Chat.

## Что получится в итоге

После настройки вы получите:

- self-hosted чат на `Rocket.Chat`
- каналы и приватные комнаты для команды до `50` человек
- групповые звонки через self-hosted `Jitsi`
- единый вход через `GitLab`
- уведомления из `GitLab` в каналы Rocket.Chat

## Порядок внедрения

Рекомендую внедрять именно в таком порядке:

1. Поднять `Rocket.Chat`
2. Проверить регистрацию администратора и работу каналов
3. Поднять `Jitsi`
4. Подключить `Jitsi` в Rocket.Chat
5. Настроить `GitLab SSO`
6. Настроить `GitLab webhook`
7. Провести пилот на 5-10 пользователях
8. После этого заводить всю команду

## Ограничения и риски

Важно учитывать:

- `PostgreSQL` как основная БД для Rocket.Chat не поддерживается
- качество видеозвонков зависит в основном от `Jitsi`, сети и открытого `10000/udp`
- если у вас много одновременных видеозвонков, узкое место обычно не Rocket.Chat, а именно `Jitsi`
- при обновлениях Rocket.Chat нужно проверять совместимость версии с `MongoDB`
- лучше не использовать `latest` в production

## Когда лучше остаться на Mattermost

Имеет смысл не переходить на Rocket.Chat и оставить `Mattermost`, если для вас критично:

- использовать именно `PostgreSQL`
- иметь более простой стек без `MongoDB`
- держать чат и звонки ближе к уже существующей инфраструктуре в этом репозитории

## Краткий вывод

Под ваш сценарий `Rocket.Chat` в бесплатной self-hosted схеме подходит.

Рекомендованный production-вариант:

- `Rocket.Chat` в Docker
- `MongoDB` как штатная БД Rocket.Chat
- `Jitsi` для звонков
- `GitLab OAuth/OIDC` для SSO
- `GitLab webhook` для событий CI/CD и разработки

Если же условие "`нужен именно PostgreSQL`" не обсуждается, тогда правильнее выбирать не `Rocket.Chat`, а `Mattermost`.

## Полезные ссылки

- Rocket.Chat installation: [https://docs.rocket.chat/docs/installation](https://docs.rocket.chat/docs/installation)
- Rocket.Chat Docker Compose: [https://docs.rocket.chat/installation/docker-containers/docker-compose](https://docs.rocket.chat/installation/docker-containers/docker-compose)
- Rocket.Chat OAuth: [https://docs.rocket.chat/use-rocket.chat/authentication/oauth](https://docs.rocket.chat/use-rocket.chat/authentication/oauth)
- Rocket.Chat GitLab integration: [https://docs.rocket.chat/use-rocket.chat/workspace-administration/integrations/gitlab](https://docs.rocket.chat/use-rocket.chat/workspace-administration/integrations/gitlab)
- Rocket.Chat Jitsi app: [https://docs.rocket.chat/docs/conference-call-admin-guide-jitsi-app](https://docs.rocket.chat/docs/conference-call-admin-guide-jitsi-app)
- GitLab as OpenID Connect provider: [https://docs.gitlab.com/integration/openid_connect_provider](https://docs.gitlab.com/integration/openid_connect_provider)
- Jitsi Docker deployment: [https://github.com/jitsi/docker-jitsi-meet](https://github.com/jitsi/docker-jitsi-meet)
