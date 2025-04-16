# Podman, Quadlet, Rathole

Заметки инфы, которая забудется. Кратко, в стиле ключевых слов (задумывалось, но не получилось, т.к. со всяким
столкнулся).

Что хочу:  
сервисы, которые крутятся на домашнем сервере, доступ снаружи через существующий VPS.

Дано:
+ Какой-то сервис в docker образах (в пример - immich)
+ Podman 5.4.2 (пока ковырялся - уже обновился с 5.2.5)
+ Rathole
+ второй ПК как сервер, но внутри домашней сети (Fedora 41)
+ бледный VPS, но где-то наруже с белым IP (Cent OS 9 Stream)

> [!NOTE]
> Считаем, что VPS уже есть, Podman, Caddy, Rathole и др. необходимый софт установлены, подняты и настроены базово
---

Можно настроить через [[Kube]](https://docs.podman.io/en/v5.2.5/markdown/podman-systemd.unit.5.html#kube-units-kube) с
Kubernetes YAML, но буду делать через `.container` и `.pod`

### Подготовка Podman quadlets

> rootless

|    | path                                 |
|----|--------------------------------------|
| or | ~/.config/containers/systemd/        |
| or | $XDG_CONFIG_HOME/containers/systemd/ |

` cd $HOME/.config/containers/systemd/ `  
` nano immich_machine_learning.container ` создаю quadlet файл контейнера

Для генерации quadlet файлов из
docker-compose.yml - [ podlet compose]( https://github.com/containers/podlet?tab=readme-ov-file#compose ).
> [!WARNING]
> генерирует включая то что можно не прописывать и были бы взяты дефолтные значения или из image, или иногда не генерирует
> что-то несколько специфичное, например, не работает с compose interpolation (см.
> примечание [ podlet compose note ](https://github.com/containers/podlet?tab=readme-ov-file#notes)). Так же падает в
> ошибку от healthcheck и т.д. - пока не поддерживается всё

т.к. в `docker-compose.yml` immich'а есть много `${VARIABLES}` и согласно комментариям в нём лучше ничего не трогать:

+ подготавливаю сервис без quadlet, "по-обычному" через docker-compose.yml
+ поднимаю `podman-compose up -d`
+ проверяю что всё работает с условно-дефолтными настройками
+ имея рабочие (не quadlet) контейнеры генерирую из них
  через[ podlet generate from existing ]( https://github.com/containers/podlet?tab=readme-ov-file#generate-from-existing ):

`podlet generate container 3fc76238438еa9` где `3fc76238438еa9`, конечно, ID контейнера

+ Проверяю вывод, правлю.

Например, сгенерированный

```
# immich_machine_learning.container
[Container]
ContainerName=immich_machine_learning
Environment=UPLOAD_LOCATION=/mah/path/immich/library DB_DATA_LOCATION=/mah/path/immich/postgres IMMICH_VERSION=release DB_PASSWORD=oopsIDidItAgain DB_USERNAME=postgres DB_DATABASE_NAME=immich
Image=ghcr.io/immich-app/immich-machine-learning:release
Label=io.podman.compose.config-hash=556c289a00c3305423dfeb47c978135f1baeed80b868184cd5ff38cdd0d881ba io.podman.compose.project=immich io.podman.compose.version=1.2.0 PODMAN_SYSTEMD_UNIT=podman-compose@immich.service com.docker.compose.project=immich com.docker.compose.project.working_dir=/mah/path/docker/immich-app com.docker.compose.project.config_files=docker-compose.yml com.docker.compose.container-number=1 com.docker.compose.service=immich-machine-learning
Network=immich_default
PodmanArgs=--network-alias immich-machine-learning --pod pod_immich
Volume=immich_model-cache:/cache
```

Можно исключить почти всё, что нет в ` docker-compose.yml ` или ` .env ` (мб каких-то ещё специфичных)
Например, не беру в quadlet'ы всякие
`Label=` - берутся с данных image контейнера.

Вместо списка `Environment=...` можно прямо использовать ` EnvironmentFile=immich.env `, предварительно скопировав его
в ` $HOME/.config/containers/systemd/ `

Вместо ` PodmanArgs=--network-alias immich-machine-learning --pod pod_immich ` (а можно оставить и так с последующими
поправками конфигов на это)
использую существующие параметры для quadlet'ов
` NetworkAlias=immich-machine-learning `

> [!NOTE]
> По идее, Network=, вроде как, можно указать только в поде, не дублируя в контейнерах, но что-то не сработало.

Вместо ` --pod pod_immich ` прописываю имя будущего пода, к которому будет относиться контейнер:
` Pod=immich.pod `

` Volume=${ML_CACHE_LOCATION}:/cache:Z ` прописываю свой путь и добавляю метку для SELinux - ` :Z `

`${ML_CACHE_LOCATION}` берётся из `EnvironmentFile=` раздела `[Service]`. Пояснение этого будет с другим контейнером.

`AutoUpdate=registry` - можно прописать в quadlet для автообновление контейнера (приложения), а можно вручную
`podman pull ghcr.io/path/address`.

Для перезапуска сервиса при ошибках и вообще

```
[Service]
Restart=always
```

Полный конфиг podman systemd quadlet подправленный

```
# immich_machine_learning.container
[Container]
ContainerName=immich_machine_learning 
EnvironmentFile=immich.env
Image=ghcr.io/immich-app/immich-machine-learning:release
Network=immich.network
NetworkAlias=immich-machine-learning
AutoUpdate=registry
Pod=immich.pod
Volume=${ML_CACHE_LOCATION}:/cache:Z

[Service]
Restart=always
EnvironmentFile=/mahHomePath/.config/containers/systemd/immich.env
```

следующий контейнер, `podlet generate container 43c568980984`

```
# immich_server.container
[Container]
AddDevice=/dev/dri:/dev/dri
ContainerName=immich_server
Environment=UPLOAD_LOCATION=/mah/path/immich/library DB_DATA_LOCATION=/mah/path/immich/postgres IMMICH_VERSION=release DB_PASSWORD=oopsIDidItAgain DB_USERNAME=postgres DB_DATABASE_NAME=immich
Image=ghcr.io/immich-app/immich-server:release
Label=io.podman.compose.config-hash=556c289a00c3305423dfeb47c978135f1baeed80b868184cd5ff38cdd0d881ba io.podman.compose.project=immich io.podman.compose.version=1.2.0 PODMAN_SYSTEMD_UNIT=podman-compose@immich.service com.docker.compose.project=immich com.docker.compose.project.working_dir=/home/koshon/docker/immich-app com.docker.compose.project.config_files=docker-compose.yml com.docker.compose.container-number=1 com.docker.compose.service=immich-server
Network=immich_default
PodmanArgs=--network-alias immich-server --pod pod_immich --requires 'immich_redis,immich_postgres'
PublishPort=2283:2283
Volume=/mah/path/immich/library:/usr/src/app/upload:Z
Volume=/etc/localtime:/etc/localtime:ro

[Service]
Restart=always
```

quadlet подправленный

```
# immich_server.container
[Unit]
Requires=immich_redis.service immich_postgres.service
After=immich_redis.service immich_postgres.service

[Container]
AddDevice=/dev/dri:/dev/dri
ContainerName=immich_server
EnvironmentFile=immich.env
Image=ghcr.io/immich-app/immich-server:release
AutoUpdate=registry
Network=immich_default
NetworkAlias=immich-server
Pod=immich.pod
Volume=/mah/path/immich/library:/usr/src/app/upload:Z
Volume=/etc/localtime:/etc/localtime:ro
PublishPort=127.0.0.1:2283:2283

[Service]
Restart=always
```

`PublishPort=127.0.0.1:2283:2283` <- 127.0.0.1 т.к. будет пробрасываться через rathole

Помимо правок как для предыдущего контейнера - добавлены зависимости от сервисов, конфиги которых будут дальше

```
[Unit]
Requires=immich_redis.service immich_postgres.service
After=immich_redis.service immich_postgres.service
```

`AddDevice=/dev/dri:/dev/dri` - не знаю, вроде проброс видеокарты для ML immich_machine_learning.container, для меня не
важно.

Следующие

сгенерированный

```
# immich_redis.container
[Container]
ContainerName=immich_redis
Image=docker.io/redis:6.2-alpine@sha256:2ba50e1ac3a0ea17b736ce9db2b0a9f6f8b85d4c27d5f5accc6a416d8f42c6d5
Label=io.podman.compose.config-hash=52d98d527eef3682fd6022ef2668d3ed4af73790b9278b9b227c9521973db760 io.podman.compose.project=immich io.podman.compose.version=1.2.0 PODMAN_SYSTEMD_UNIT=podman-compose@immich.service com.docker.compose.project=immich com.docker.compose.project.working_dir=/mah/path/docker/immich-app com.docker.compose.project.config_files=docker-compose.yml com.docker.compose.container-number=1 com.docker.compose.service=redis
Network=immich_default
PodmanArgs=--network-alias redis --pod pod_immich
Volume=/mah/path/immich/redis_data:/data:Z

[Service]
Restart=always
```

Подправленный

```
# immich_redis.container
[Container]
ContainerName=immich_redis
Image=docker.io/redis:6.2-alpine@sha256:2ba50e1ac3a0ea17b736ce9db2b0a9f6f8b85d4c27d5f5a>
Network=immich_default
NetworkAlias=redis
HealthCmd=redis-cli ping || exit 1
Pod=immich.pod
Volume=/mah/path/immich/redis_data:/data:Z

[Service]
Restart=always
```

> [!NOTE]
> удалял с параметров контейнеров в docker-compose.yml строчки относящиеся к health, иначе podlet падает с ошибкой. Пока
> прописываю вручную health options quadlet'ов.

Сгенерированный

```
# immich_postgres.container
[Container]
ContainerName=immich_postgres
Image=docker.io/tensorchord/pgvecto-rs:pg14-v0.2.0@sha256:739cdd626151ff1f796dc95a6591b55a714f341c737e27f045019ceabf8e8c52
Environment=POSTGRES_PASSWORD=oopsIDidItAgain POSTGRES_USER=postgres POSTGRES_DB=immich POSTGRES_INITDB_ARGS=--data-checksums
Exec=postgres -c 'shared_preload_libraries=vectors.so' -c 'search_path="$user", public, vectors' -c 'logging_collector=on' -c 'max_wal_size=2GB' -c 'shared_buffers=512MB' -c 'wal_compression=on'
Label=io.podman.compose.config-hash=a0034e62618b2ee44cf0f412eb24dbfecaa7a0a6276db0ed831c49951b2a267a io.podman.compose.project=immich io.podman.compose.version=1.2.0 PODMAN_SYSTEMD_UNIT=podman-compose@immich.service com.docker.compose.project=immich com.docker.compose.project.working_dir=/mah/path/docker/immich-app com.docker.compose.project.config_files=docker-compose.yml com.docker.compose.container-number=1 com.docker.compose.service=database
Network=immich_default
PodmanArgs=--network-alias database --pod pod_immich
Volume=/mah/path/immich/postgres:/var/lib/postgresql/data:Z
```

Подправленный

```
# immich_postgres.container
[Service]
EnvironmentFile=/mahHomePath/.config/containers/systemd/immich.env
Restart=always

[Container]
ContainerName=immich_postgres
Image=docker.io/tensorchord/pgvecto-rs:pg14-v0.2.0@sha256:739cdd626151ff1f796dc95a6591b55a714f341c737e27f045019ceabf8e8c52
Environment=POSTGRES_PASSWORD=${DB_PASSWORD}
Environment=POSTGRES_USER=${DB_USERNAME}
Environment=POSTGRES_DB=${DB_DATABASE_NAME}
Environment=POSTGRES_INITDB_ARGS='--data-checksums'
Exec=postgres -c 'shared_preload_libraries=vectors.so' -c 'search_path="$user", public, vectors' -c 'logging_collector=on' -c 'max_wal_size=2GB' -c 'shared_buffers=512MB' -c 'wal_compression=on'
HealthCmd=/bin/sh -c 'pg_isready --dbname="${DB_DATABASE_NAME}" --username="${DB_USERNAME}" || exit 1; Chksum=$(psql --dbname="${DB_DATABASE_NAME}" --username="${DB_USERNAME}" --tuples-only --no-align --comman>
HealthInterval=5m
HealthStartPeriod=5m
HealthStartupInterval=30s
Network=immich_default
NetworkAlias=database
Pod=immich.pod
Volume=/mah/path/immich/postgres:/var/lib/postgresql/data:Z
```

> [!NOTE]
> ```
> Environment=POSTGRES_PASSWORD=${DB_PASSWORD}
> Environment=POSTGRES_USER=${DB_USERNAME}
> Environment=POSTGRES_DB=${DB_DATABASE_NAME}
> ```
> (!) в таком виде переменные не передаются через `EnvironmentFile=immich.env` внутри раздела `[Container]`, т.к. это
> только
> для переменных внутри контейнера.
> Поэтому для quadlet'а передаю через раздел `[Service]` как systemd переменные. (!) обязательно полный путь
>``` 
> [Service]
> EnvironmentFile=/mahHomePath/.config/containers/systemd/immich.env
> ```

`.env` файл представляет собой что-то вроде
```
#IMMICH_HOST=

# The location where your uploaded files are stored
UPLOAD_LOCATION=/mah/path/immich/library
# The location where your database files are stored
DB_DATA_LOCATION=/mah/path/immich/postgres
# ML
ML_CACHE_LOCATION=/mah/path/immich/immich_model-cache
# REDIS
REDIS_DATA=/mah/path/immich/redis_data

# To set a timezone, uncomment the next line and change Etc/UTC to a TZ identifier from this list: https://en.wikipedia.org/wiki/List_of_tz_database_time_zones#List
# TZ=Etc/UTC

# The Immich version to use. You can pin this to a specific version like "v1.71.0"
IMMICH_VERSION=release

# Connection secret for postgres. You should change it to a random password
# Please use only the characters `A-Za-z0-9`, without special characters or spaces
DB_PASSWORD=oopsIDidItAgain

# The values below this line do not need to be changed
###################################################################################
DB_USERNAME=postgres
DB_DATABASE_NAME=immich
```

Собираем `.pod`

Можно создать свой `.network` quadlet файл со своей спецификой, но использую уже существующий, который был создан при
обычном `podman-compose up -d` и `docker-compose.yml`

```
[koshon@fedora]$ podman network ps
...
1b584399e51d  immich_default             bridge
...

```

```
[koshon@fedora]$ podman network inspect immich_default
[
     {
          "name": "immich_default",
          "id": "9da7c73a984b6cbe0ada4843added527f7d2937ef844ff452940ee70cd89a1c0",
          "driver": "bridge",
          "network_interface": "podman16",
          "created": "2024-12-02T20:55:51.293782218+04:00",
          "subnets": [
               {
                    "subnet": "10.89.15.0/24",
                    "gateway": "10.89.15.1"
               }
          ],
          "ipv6_enabled": false,
          "internal": false,
          "dns_enabled": true,
          "labels": {
               "com.docker.compose.project": "immich",
               "io.podman.compose.project": "immich"
          },
          "ipam_options": {
               "driver": "host-local"
          },
          "containers": {}
     }
]
```

```
# immich.pod

[Pod]
PodName=immich-pod
PublishPort=127.0.0.1:2283:2283
Network=immich_default
NetworkAlias=immich-pod
```

`PublishPort=127.0.0.1:2283:2283` использую внутри, т.к. наружу будет проброшено через rathole.
Дополнительно для себя пробрасываю всё под хостовыми UID/GID `UserNS=keep-id:uid=1000,gid=1000`. Обычно достаточно
прописать это только в поде, если не нужна какая-то специфика для какого-то контейнера или нескольких.

Прогоняю через внутренний механизм проверки синтаксиса podman'ом создания quadlet systemd файлов

`/usr/lib/systemd/system-generators/podman-system-generator --user --dryrun` или
`/usr/lib/systemd/system-generators/podman-system-generator -user -dryrun` или лучше
`systemd-analyze --user --generators=true verify immich-pod.service`

[Ещё вариант](https://docs.podman.io/en/v5.2.5/markdown/podman-systemd.unit.5.html#debugging-a-limited-set-of-unit-files),
чтобы дебажить только редактируемые quadlet'ы, но по мне достаточно `/usr/lib/systemd/system-generators/podman-system-generator -user -dryrun 2>&1 | grep -i "error"`

Пришлось создать специально "тупой" файл для проверки работы чекера синтаксиса юнита, т.к. практически любые намеренные
ошибки в исходных quadlet'ах не выдавало никакой обратной связи.

```
# immich_error.container
p[Unit]
```

```
[koshon@fedora]$ /usr/lib/systemd/system-generators/podman-system-generator -user -dryrun 2>&1 | grep -i "error"
quadlet-generator[23975]: Loading source unit file /home/koshon/.config/containers/systemd/immich_error.container
quadlet-generator[23975]: error loading "/home/koshon/.config/containers/systemd/immich_error.container", file contains line 1: “p[Unit]” which is not a key-value pair, group, or comment
```

Перезагружаю пользовательский systemd сервис `systemctl --user daemon-reload`
> [!NOTE]
> `systemctl --user daemon-reload` после внесения ЛЮБЫХ изменений в quadlet systemd файлы!

Список пользовательских юнитов `systemctl --user list-unit-files`

Запускаю сервис от пользователя `systemctl --user start immich-pod.service`

Генерирую токен для rathole'а `openssl rand -base64 32`, который будет использоваться для проверки подлинности между
rathole на хосте и rathole на сервере. Можно хоть 12345 поставить, не обязательно base64.

Конфигурационные файлы rathole сделаны через [systemd](https://github.com/rapiz1/rathole/tree/main/examples/systemd)

на хосте в `/etc/systemd/system/` `ratholec@.service` ('c' в ratholec потому, что client)

```
[Unit]
Description=Rathole Client Service
After=network.target

[Service]
Type=simple
Restart=on-failure
RestartSec=5s
LimitNOFILE=1048576

# with root
ExecStart=/usr/local/bin/rathole -c /etc/rathole/%i.toml
# without root
# ExecStart=%h/.local/bin/rathole -c %h/.local/etc/rathole/%i.toml

[Install]
WantedBy=multi-user.target
```

в `/etc/rathole/` `immich.toml`

```
# client.toml
[client]
# В конфигах rathole'a, если он настроен через systemd и multiple instances (@) - прописывать свободные/разные порты rathole'а, а не один и тот же.
remote_addr = "mah_vps_address:2334" # или IP:port. Порт не сервиса в контейнере, а rathole'а на сервере!

[client.services.immich]
type = "tcp" # протокол сервиса, который пробрасываем
token = "mah_generated_token="
local_addr = "127.0.0.1:2283" # адрес:порт пробрасываемого сервиса на хосте
```

Запускаю rathole systemd service для immich `sudo systemctl enable ratholec@immich --now`

Настройка rathole на стороне vps'а/сервера.

в `/etc/systemd/system/` `ratholes@.service` ('s' в ratholes потому, что server)

```
[Unit]
Description=Rathole Server Service
After=network.target

[Service]
Type=simple
Restart=on-failure
RestartSec=5s
LimitNOFILE=1048576

# with root
ExecStart=/usr/local/bin/rathole -s /etc/rathole/%i.toml
# without root
# ExecStart=%h/.local/bin/rathole -s %h/.local/etc/rathole/%i.toml

[Install]
WantedBy=multi-user.target
```

Разница между хостом и сервером в параметрах запуска бинарника rathole'а: -c или -s

в `/etc/rathole/` `immich.toml`

```
# server.toml
[server]
bind_addr = "0.0.0.0:2334"

[server.services.immich]
type = "tcp" # так же как у client
token = "mah_generated_token="
bind_addr = "127.0.0.1:2283" # на какой порт vps'а/сервера подвязываем. Т.е. это последняя часть в цепочке local_container_host:port <-> rathole <-> global_vps:port
```

> [!NOTE]
>
> Настройки Caddy через /путь с immich не работают. Следующее - нерабочий вариант, который, впрочем, вполне себе
> работает с другими сервисами
> ```
> mah_vps.address {
>       	handle /i* {
>               	redir /immich /immich/
>               	uri strip_prefix /immich
>               	reverse_proxy localhost:2283
>       	}
> }
> ```

Дальше уже рабочая часть Caddyfile через subdomain, которая работает с immich

```
subdomain_for_immich.mah_vps.address {
       	reverse_proxy localhost:12283
}
```

проверяем конфиг `sudo caddy validate --config /etc/caddy/Caddyfile`, если нужно форматирую
`sudo caddy fmt --overwrite /etc/caddy/Caddyfile`, применяем изменения `sudo systemctl reload caddy` (или если отключен
`admin off` в глобальном блоке настроек Caddyfile'а `sudo systemctl restart caddy`), заходим на свой
`subdomain_for_immich.mah_vps.address` - видим вход на admin панель.
Занавес.

---

> [!Note] Всякие заметки
>
> Если нужна большая гибкость - не писать параметры в поде, а заполнять конкретно в каждом контейнере.
> 
> Разрабы podman только к версии ~5.4 поняли, что скачивание образа может прерваться, не докачаться и стать битым. Уму
> не постижимо, как до этого времени ошибки не всплывало / не было обратной связи. На 5.4.2 пока так что и падает с
> каким-то из битых layer'ов - перепроверять всё пришлось на домашнем компе, а не домашнем сервере, где пока так и не понятно
> как почистить так, чтобы это всё нафиг заработало! Что прунь, что не прунь, что вручную удаляй почти всё - постоянный крэш (и не тест, и не любовь!)!
>
> Не знал, менял названия в `NetworkAlias=` на свои предпочтения и получал ошибку по
`aardvark-dns dns request got empty response`.
>
> Плюсом ещё одна ошибка при неправильно заполненном `NetworkAlias=`
>
> ```
> 22:37 } immich_server
> 22:37 syscall: 'connect' immich_server
> 22:37 code: 'ETIMEDOUT', immich_server
> 22:37 errorno: 'ETIMEDOUT', immich_server
> 22:37 at process.processTimers (node:internal/timers:529:7) { immich_server
> 22:37 at listOnTimeout (node:internal/timers:594:17) immich_server
> 22:37 at Socket._onTimeout (node:net:595:8) immich_server
> 22:37 at Socket.emit (node:events:518:28) immich_server
> 22:37 at Object.onceWrapper (node:events:632:28) immich_server
> 22:37 at Socket.<anonymous> (/usr/src/app/node_modules/ioredis/built/Redis.js:170:41) immich_server
> 22:37 Error: connect ETIMEDOUT immich_server
> 22:37 } immich_server
> } 
> ```
> или похожая ошибка по redis - короче, связано с `NetworkAlias=`
>
> `PublishPort=2283:2283` прописывать в `immich_server.container`, а не в `immich.pod`. Хз почему так, но в этом конкретном
> случае так.
>
> Обязательно в следующий раз использовать `podman inspect ID` на контейнерах созданных через `docker-compose.yml` с
> созданными через quadlets
> для сравнения различий параметров с помощью `diff`.
>
> [Наткнулся](https://github.com/tbelway/immich-podman-quadlets/blob/main/docs/install/podman-quadlet.md) уже после
> самоковыряний, но целью и не было настроить по гайду, а медленно пройтись по граблям, вкушая осинку головой и понять
> как работают quadlet'ы и какие есть нюансы.
>
> ```[Unit]
> Wants=network-online.target
> After=network-online.target
> Requires=container-container0.service container-container1.service
> Before=container-container0.service container-container1.service
> BindsTo=pod-systemd-pod.service
> ```
>
>     [Unit]:
>         Этот раздел определяет общие параметры юнита.
>     Wants=network-online.target:
>         Указывает, что этот юнит "желает" (но не требует) запуска network-online.target. Это означает, что юнит будет запущен после того, как сеть станет доступной, если это возможно, но не будет ожидать, если сеть не готова.
>     After=network-online.target:
>         Указывает, что этот юнит должен быть запущен после того, как network-online.target будет запущен. Это гарантирует, что сеть будет доступна перед запуском этого юнита.
>     Requires=container-container0.service container-container1.service:
>         Указывает, что этот юнит требует запуска container-container0.service и container-container1.service. Если эти службы не могут быть запущены, этот юнит также не будет запущен.
>     Before=container-container0.service container-container1.service:
>         Указывает, что эти сервисы должны быть запущены после запуска данного юнита. Это может быть полезно для того, чтобы задать определенный порядок запуска сервисов.
>     BindsTo=pod-systemd-pod.service:
>         Указывает, что этот юнит "привязан" к pod-systemd-pod.service. Это означает, что если pod-systemd-pod.service завершит работу, этот юнит также будет остановлен.
>
> quadlet переименовывает файлы при переводе в systemd. Например, `immich.network` становится `immich-network.service`. Для
> исключения конфликтов когда `immich.pod` и какой-то контейнер могли бы оба быть `immich.service`.
>
> `NetworkAlias=` в контейнере не работает без указания `Network=`. Если контейнер в поде - смысла прописывать оба нет.