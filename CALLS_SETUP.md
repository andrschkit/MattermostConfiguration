# Mattermost Calls Setup

Этот документ описывает настройку встроенного `Mattermost Calls` для вашей схемы:

- внешний адрес чата: `https://chat.uaz-rosatom.ru`
- внешний `nginx` уже проксирует HTTP/WebSocket на `192.168.10.80:8065`
- сам `Mattermost` работает на отдельной ВМ `192.168.10.80`

В данном репозитории используется встроенный режим `Calls plugin (integrated mode)` без отдельного `RTCD`.

## Что уже добавлено в docker-compose

В `docker-compose.yaml` для контейнера `mattermost` уже настроены:

- публикация `8443/udp` для медиатрафика звонков
- публикация `8443/tcp` как fallback
- переменные `MM_CALLS_UDP_SERVER_PORT` и `MM_CALLS_TCP_SERVER_PORT`
- переменные `MM_CALLS_ICE_HOST_OVERRIDE` и `MM_CALLS_ICE_HOST_PORT_OVERRIDE`

## Что нужно указать в .env

Минимально:

```env
CALLS_UDP_PORT=8443
CALLS_TCP_PORT=8443
CALLS_ICE_HOST_OVERRIDE=PUBLIC_IP_OR_FQDN
CALLS_ICE_HOST_PORT_OVERRIDE=8443
CALLS_DEFAULT_ENABLED=true
CALLS_ALLOW_ENABLE_CALLS=true
```

Рекомендации по `CALLS_ICE_HOST_OVERRIDE`:

- если `chat.uaz-rosatom.ru` резолвится в тот же внешний адрес, на котором будет доступен `8443`, можно указать `chat.uaz-rosatom.ru`
- если внешняя балансировка или `DNS` сложные, надежнее указать внешний публичный `IP`
- не указывайте внутренний адрес вроде `192.168.10.80`, иначе клиент снаружи не сможет подключиться

## Что обязательно настроить в сети

Обычного HTTP reverse proxy через `nginx` недостаточно для звонков.

Для `Calls` клиенты должны иметь прямой доступ к медиа-порту `8443`:

- `UDP 8443` с клиентов до ВМ `Mattermost`
- `TCP 8443` с клиентов до ВМ `Mattermost`

В вашей схеме это означает одно из двух:

1. На внешнем контуре настроить `DNAT/port forwarding` с публичного адреса на `192.168.10.80:8443` для `UDP` и `TCP`.
2. Либо настроить отдельную `L4`-маршрутизацию для `8443`, если она у вас делается не через `DNAT`.

Важно:

- обычный `nginx`-proxy для сайта на `443 -> 8065` не проксирует звонки автоматически
- для звонков недостаточно только `443/tcp`
- если `UDP 8443` закрыт, звонки часто не поднимаются или работают нестабильно

## Что проверить на Mattermost VM

На ВМ `192.168.10.80` должны быть доступны:

- `8065/tcp` для самого `Mattermost`
- `8443/udp` для `Calls`
- `8443/tcp` для fallback

Также проверьте локальный firewall ОС и сетевые ACL между пользователями и ВМ.

## Что включить в UI Mattermost

Зайдите под системным администратором:

1. `System Console`
2. `Plugins`
3. `Calls`

Проверьте настройки:

- `Enable plugin` = `true`
- `Test Mode` выключен, если хотите дать право запускать звонки обычным пользователям
- `ICE Host Override` соответствует вашему внешнему адресу
- `RTC Server Port (UDP)` = `8443`
- `RTC Server Port (TCP)` = `8443`

Если часть настроек уже задана через переменные окружения, в UI они могут быть доступны только для просмотра.

## Когда нужен TURN

Если пользователи подключаются из внешних сетей с жесткими ограничениями, через VPN, мобильные сети или корпоративные прокси, одного `8443` может быть недостаточно.

В таком случае потребуется `TURN` сервер. Для первого этапа рекомендую:

- сначала проверить звонки внутри вашей сети с открытым `8443/udp`
- если проблема сохраняется только у части пользователей, уже потом добавлять `TURN`

## Проверка после запуска

1. Перезапустить стек:

```bash
docker compose up -d
```

2. Убедиться, что контейнер слушает `8443`:

```bash
docker ps
```

3. Создать тестовый канал и начать звонок
4. Проверить, появляется ли подключение без ошибки `unable to connect the voice call`

Если ошибка остается:

- проверьте, что `CALLS_ICE_HOST_OVERRIDE` указывает на внешний адрес, доступный клиентам
- проверьте доступность `8443/udp` и `8443/tcp`
- проверьте, не блокирует ли трафик firewall или NAT
- соберите клиентские логи через `/call logs`

## Официальная документация

- [Deploy Mattermost Calls](https://docs.mattermost.com/administration-guide/configure/calls-deployment.html)
- [Plugins Configuration Settings: Calls](https://docs.mattermost.com/administration-guide/configure/plugins-configuration-settings.html#calls)
- [Troubleshooting Mattermost Calls](https://docs.mattermost.com/administration-guide/configure/calls-troubleshooting.html)
- [Make Calls](https://docs.mattermost.com/end-user-guide/collaborate/make-calls.html)

## Примечание по RTCD

Для single-node установки встроенный `Calls` обычно проще в эксплуатации. Отдельный `RTCD` имеет смысл рассматривать позже, если вам понадобятся более сложные сценарии масштабирования или выделенная media-служба.
