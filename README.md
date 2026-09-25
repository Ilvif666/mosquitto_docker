# mosquitto_docker — девять MQTT-брокеров стенда SmartWard

Каждому шкафу (инстансу бэкенда) соответствует свой брокер: один шкаф — один брокер,
один внешний порт. Брокеры — общая инфраструктура: они живут дольше бэкендов, поэтому
конфигурацию, пароли, retained-состояние и тома нельзя терять при работах с приложениями.
Если шкаф временно переведён в storage-only, его брокер остаётся в compose и может быть
поднят снова.

## Брокеры и порты

| Сервис compose | Контейнер | Порт на стенде | Порт в контейнере | Индекс брокера |
|---|---|---|---|---|
| `mosquitto_0` | `mosquitto_docker_mosquitto_0_1` | 1883 | 1883 | 0 |
| `mosquitto_1` | `mosquitto_docker_mosquitto_1_1` | 1884 | 1883 | 1 |
| `mosquitto_2` | `mosquitto_docker_mosquitto_2_1` | 1885 | 1883 | 2 |
| `mosquitto_3` | `mosquitto_docker_mosquitto_3_1` | 1886 | 1883 | 3 |
| `mosquitto_4` | `mosquitto_docker_mosquitto_4_1` | 1887 | 1883 | 4 |
| `mosquitto_5` | `mosquitto_docker_mosquitto_5_1` | 1888 | 1883 | 5 |
| `mosquitto_6` | `mosquitto_docker_mosquitto_6_1` | 1889 | 1883 | 6 |
| `mosquitto_7` | `mosquitto_docker_mosquitto_7_1` | 1890 | 1883 | 7 |
| `mosquitto_8` | `mosquitto_docker_mosquitto_8_1` | 1891 | 1883 | 8 |

Все девять сервисов используют один образ `eclipse-mosquitto:latest`, один общий
`mosquitto.conf` и один общий файл паролей, и подключены к внешней сети `sw_net`.

Соответствие «шкаф → брокер» задаётся в `/etc/smartward/instances/<instance>.env`:
индекс брокера = `INDEX − 1`, внешний порт = `1883 + (INDEX − 1)`. Примеры со стенда:
`b0` (`INDEX=1`) → `mosquitto_0`, порт 1883; `b3` (`INDEX=4`) → `mosquitto_3`, порт 1886;
`box` (`INDEX=5`) → `mosquitto_4`, порт 1887; `b5` (`INDEX=6`) → `mosquitto_5`, порт 1888.
Проверочные значения для конкретного шкафа — `BROKER_ADDRESS` и `BROKER_PORT` в том же файле.

## Требования

- Docker и **docker-compose 1.29.2** (на стенде это отдельный бинарник `docker-compose`;
  подкоманды `docker compose` нет).
- Внешняя сеть `sw_net` (`docker network create sw_net`, если её ещё нет).
- Образ `eclipse-mosquitto:latest` (тянется при первом запуске).

## Запуск и остановка

На стенде все брокеры поднимаются одним systemd-юнитом `mosquitto.service` (oneshot):

```ini
[Service]
Type=oneshot
RemainAfterExit=yes
WorkingDirectory=/home/pi/sw/mosquitto_docker
ExecStart=docker-compose up -d
ExecStop=docker-compose down
```

```bash
sudo systemctl start mosquitto      # поднять все брокеры
sudo systemctl status mosquitto     # oneshot: "active (exited)" — это норма
```

Точечные операции — из каталога `mosquitto_docker`:

```bash
docker restart mosquitto_docker_mosquitto_3_1          # перезапуск одного брокера
docker-compose up -d mosquitto_3                       # пересоздать один сервис (том с данными сохраняется)
docker logs mosquitto_docker_mosquitto_3_1 --tail 50   # логи одного брокера
```

**Не перезапускайте все брокеры ради одного шкафа.** `mosquitto.service` действует на все
девять сразу, а брокеры обслуживают разные шкафы. С 2026-09-25 данные брокера лежат в
именованном томе (`mosquitto_<индекс>_data`), поэтому пересоздание контейнера и перезагрузка
стенда retained-состояние не теряют (см. «Retained-состояние и тома»).

## Конфигурация

Все брокеры читают один и тот же `mosquitto.conf`, смонтированный файлом
(`./mosquitto.conf:/mosquitto/config/mosquitto.conf`):

| Директива | Значение | Зачем |
|---|---|---|
| `persistence` | `true` | retained-сообщения и подписки переживают перезапуск брокера |
| `persistence_location` | `/mosquitto/data/` | штатный data-каталог образа, принадлежит uid 1883 |
| `log_dest` | `stdout` (+ `topic`) | логи забирает контейнерный рантайм; брокеру не нужны права на host-каталог |
| `log_type` | `all` | полный лог для разбора инцидентов |
| `allow_anonymous` | `false` | анонимные подключения запрещены (проверено: `Connection Refused: not authorised`) |
| `listener` | `1883 0.0.0.0` | брокер слушает внутри контейнера на всех интерфейсах |
| `password_file` | `/etc/mosquitto/passwd` | файл паролей, смонтирован из репозитория |

Правки `mosquitto.conf` применяются только после перезапуска брокеров: файл читается
на старте. Директивы `acl_file` в конфигурации **нет** — разграничение доступа строится
на пароле и на том, что у каждого шкафа свой брокер. Порт публикуется на все интерфейсы
стенда, поэтому брокеры доступны любому хосту локальной сети, знающему пароль; наружу
(в интернет) их публиковать нельзя.

## Пароли

Файл `passwd` смонтирован как `/etc/mosquitto/passwd` во все брокеры. В нём сейчас одна
учётная запись — `client`, общая для всех шкафов и всех брокеров.

- Файл должен быть читаем пользователем `mosquitto` (uid 1883): рабочие права — `640`,
  владелец `1883:1883`. Если после копирования/чекаута владелец сменился на `pi`,
  брокеры не стартуют.
- Добавление и смена пароля — штатной утилитой внутри контейнера (запись идёт через
  bind-mount прямо в файл репозитория):

  ```bash
  docker exec -it mosquitto_docker_mosquitto_0_1 \
    mosquitto_passwd -b /etc/mosquitto/passwd client '<новый-пароль>'
  sudo systemctl restart mosquitto     # пароль перечитывается только при старте
  ```

  Помните: `systemctl restart mosquitto` — это глобальный `down` + `up` всех брокеров
  (см. раздел про retained), а сам пароль после такой операции нужно раздать всем шкафам.

- **Файл `passwd` отслеживается git.** Хеши паролей лежат и в истории репозитория, поэтому
  смена пароля — это ещё и отдельная задача про очистку/вынос файла из репозитория; в
  README и в переписке пароли не приводятся.

## Провижининг брокеров через Docker Manager

Брокеры не создаются руками: их добавляет и удаляет пульт (`docker_manager`).

- При создании сервиса шкафа вызывается `ensureMosquittoBroker(index)` из
  `services/serviceCreationService.js`: он дописывает в `mosquitto_docker/docker-compose.yml`
  сервис `mosquitto_<index − 1>` (шаблон — `utils/serviceTemplate.js#createMosquittoServiceConfig`:
  образ `eclipse-mosquitto:latest`, порт `1883 + индекс − 1`, монтирования `mosquitto.conf`,
  `passwd` и именованный том `mosquitto_<индекс − 1>_data` на `/mosquitto/data`, сеть `sw_net`),
  объявляет том в секции `volumes:` и запускает сервис.
- При переводе шкафа в storage-only соответствующий сервис убирается из compose
  (`removeMosquittoBrokerFromCompose`), но данные брокера и его запись в compose должны
  оставаться восстановимыми: удалять брокер только потому, что бэкенд сейчас остановлен,
  нельзя.
- Правка `docker-compose.yml` пультом — это правка файла в репозитории: изменения нужно
  коммитить и доставлять через PR, как и любые другие.

## Retained-состояние и тома

`/mosquitto/data` — data-каталог образа, в нём лежит `mosquitto.db` (retained-сообщения и
подписки); `persistence_location` в `mosquitto.conf` указывает именно туда. С 2026-09-25 этот
каталог у каждого брокера смонтирован **именованным томом**:

```yaml
    volumes:
      - ./mosquitto.conf:/mosquitto/config/mosquitto.conf
      - ./passwd:/etc/mosquitto/passwd
      - mosquitto_0_data:/mosquitto/data
```

Имена томов объявлены в секции `volumes:` того же `docker-compose.yml`
(`mosquitto_0_data` … `mosquitto_8_data`). `/mosquitto/log` в compose не объявлен: логи уходят
в stdout (`log_dest stdout`), этот каталог не используется.

**Почему так.** У образа `eclipse-mosquitto` есть `VOLUME /mosquitto/data`, поэтому без явного
объявления Docker создаёт анонимный том. Compose 1.29.2 (версия 1) анонимные тома при
пересоздании контейнера не переносит: `up -d` после правки compose или образа, `up -d
--force-recreate` и цикл `down` + `up` оставляют `/mosquitto/data` пустым. Именно это делали
`ExecStop`/`ExecStart` юнита `mosquitto.service`, поэтому retained-состояние не переживало
перезагрузку стенда (наблюдение 2026-09-25: у всех девяти брокеров каталог был пуст, а тома
имели время создания, совпадающее со стартом системы). Сохранял состояние только точечный
`docker restart`. С именованным томом данные переживают и пересоздание контейнера, и
перезагрузку стенда.

Проверено на стенде 169 (docker-compose 1.29.2, 2026-09-25) на контрольном retained-сообщении
`test/stage14/probe`: после публикации оно читается обратно и переживает все три сценария —
`docker restart` контейнера, `docker-compose up -d --force-recreate mosquitto_0` и
`sudo systemctl restart mosquitto` (полный `down` + `up` девяти брокеров, то есть то же, что
происходит при загрузке стенда). `mosquitto.db` при этом лежит в именованном томе
(`mosquitto_docker_mosquitto_<n>_data`) и растёт по мере появления retained-состояния.
Контрольное сообщение после проверки снято.

Две оговорки из той же проверки:

* `mosquitto.db` пишется не сразу: по умолчанию раз в 30 минут (`autosave_interval`) и при
  корректной остановке брокера (`SIGTERM`). Поэтому «пустой файл в томе» при работающем
  контейнере — это норма, а не потеря данных; но и остановка брокера перед переносом томов
  обязательна (см. ниже).
* снимок `mosquitto_sub -t '#' -v -W 3` **не равен** списку retained: за три секунды окна в
  него попадает и живой трафик (телеметрия устройств, команды). Для retained-состояния
  используйте `mosquitto_sub -t '#' -v -R -W 3` (`--retained-only`).

**Что именно терялось.** В бэкенде с `retain: true` публикуется команда
`commands/update/deep_sleep` (`smartWard/controllers/mqttController.js`); кроме неё retained
бывает у сообщений самих устройств (`telemetry/status`, `telemetry/weight`) и у команд
`commands/send/weight`, поэтому терялось не только deep sleep. После пересоздания брокеров
устройство не получало последнюю retained-команду и оставалось на своём состоянии из EEPROM —
если настройку нужно применить, её публикуют заново из интерфейса. Данные `unit_weight` и
данные дисплея устройства запрашивают у бэкенда сами при включении
(`backend/commands/request_unit_weight`, `backend/commands/request_display_data`), они
публикуются без `retain` и от retained-состояния брокера не зависят.

**Перенос данных со старых анонимных томов** (так это делалось на стенде 169 2026-09-25;
бэкап — `/home/pi/backup/mqtt-anon-<метка>/`):

1. выписать монтирования и скопировать содержимое анонимных томов в бэкап:
   `docker inspect -f '{{json .Mounts}}' mosquitto_docker_mosquitto_<n>_1`, затем
   `docker exec mosquitto_docker_mosquitto_<n>_1 tar -C /mosquitto -cf - data log > <бэкап>.tar`;
2. **остановить брокеров, а не удалять их**: `docker-compose stop` — по `SIGTERM` mosquitto
   записывает `mosquitto.db` в текущий анонимный том (иначе состояние, живущее только в памяти,
   будет потеряно при пересоздании);
3. обновить `docker-compose.yml`, создать именованные тома и перенести в них данные (образ
   `alpine` не нужен, подходит сам `eclipse-mosquitto`):

   ```bash
   docker volume create mosquitto_docker_mosquitto_<n>_data
   docker run --rm -u 0 --entrypoint sh \
     --volumes-from mosquitto_docker_mosquitto_<n>_1 \
     -v mosquitto_docker_mosquitto_<n>_data:/to eclipse-mosquitto:latest \
     -c 'cp -a /mosquitto/data/. /to/ && chown -R 1883:1883 /to'
   ```

4. поднять брокеров заново: `docker-compose up -d` — контейнеры пересоздаются, но данные уже
   лежат в именованных томах;
5. проверить: `docker inspect` показывает `mosquitto_docker_mosquitto_<n>_data` на
   `/mosquitto/data`, retained на месте (`mosquitto_sub -t '#' -v -R -W 3`), и после
   `sudo systemctl restart mosquitto` retained не изменился.

Осиротевшие анонимные тома после проверки можно удалить, чтобы не занимать место, но только
осознанно (`docker volume ls -qf dangling=true`) и с согласия владельца стенда.
`docker-compose down -v` в этом каталоге по-прежнему недопустим: `-v` удаляет и именованные
тома вместе с данными брокеров.

## Логи

Логи брокеров уходят в stdout и забираются контейнерным рантаймом:

```bash
docker logs mosquitto_docker_mosquitto_0_1 --tail 100
docker logs mosquitto_docker_mosquitto_0_1 --since 10m | grep -i error
```

Юнит `mosquitto.service` — oneshot, поэтому в `journalctl -u mosquitto` логов самих
брокеров нет: смотреть нужно логи контейнеров.

**Почему нельзя вернуть монтирование host-каталога логов.** В прежнем варианте compose
каталог логов монтировался из репозитория (`./mosquitto_logs/<файл>:/var/log/mosquitto/...`).
Путь-файл превратился в каталог, созданный от root, а брокер работает под uid 1883 и не мог
туда писать; попытки autosave печатали `No such file or directory`. Каталог
`mosquitto_docker/mosquitto_logs/` на стенде — остаток того варианта (root:root,
подкаталоги `ward*.log`); он не монтируется и не используется. Возвращать этот маунт не
нужно.

## Проверка и диагностика

```bash
# слышит ли конкретный брокер свой порт
docker exec mosquitto_docker_mosquitto_0_1 sh -c 'netstat -ltn | head'

# анонимное подключение должно быть отвергнуто
docker exec mosquitto_docker_mosquitto_0_1 mosquitto_sub -p 1883 -t 'test/anon' -C 1 -W 2
# → Connection error: Connection Refused: not authorised.

# рабочий обмен сообщениями (пароль — из /etc/smartward, в репозитории его нет)
docker exec -it mosquitto_docker_mosquitto_0_1 \
  mosquitto_sub -p 1883 -u client -P '<пароль>' -t 'sw169/selftest' -C 1 -W 5
docker exec -it mosquitto_docker_mosquitto_0_1 \
  mosquitto_pub -p 1883 -u client -P '<пароль>' -t 'sw169/selftest' -m ok
```

Проверять нужно именно содержимое (сообщение дошло, retained читается после перезапуска
брокера), а не только код ответа: в этом стенде уже дважды «зелёные» проверки скрывали
неверный артефакт.

## Грабли

1. **`docker-compose down` сам по себе больше не стирает retained-состояние** — с 2026-09-25
   данные лежат в именованном томе (см. раздел про тома). Но `docker-compose down -v`, удаление
   тома или возврат к compose без объявленных томов состояние по-прежнему уничтожат.
2. **Один общий `mosquitto.conf` и один общий `passwd`** на девять брокеров: правка
   затрагивает все шкафы, перезапуск — тоже.
3. **Права на `passwd`.** Файл должен читаться uid 1883; иначе брокеры не стартуют.
4. **Пароль один на всех** (`client`), и его хеш лежит в git — при ротации нужно обновить
   файл и перезапустить брокеров, а сам файл вынести из репозитория.
5. **Не удалять брокер остановленного шкафа** и не удалять тома данных: они нужны при
   возврате шкафа в работу.
6. **CRLF.** При правке `docker-compose.yml` и `mosquitto.conf` следите за переводами строк
   (`git diff --ignore-cr-at-eol`), иначе PR выглядит как полная перезапись файла.
