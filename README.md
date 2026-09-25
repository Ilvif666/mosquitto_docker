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
docker restart mosquitto_docker_mosquitto_3_1          # перезапуск одного брокера (состояние сохраняется)
docker-compose up -d mosquitto_3                       # пересоздать один сервис (retained-состояние будет потеряно)
docker logs mosquitto_docker_mosquitto_3_1 --tail 50   # логи одного брокера
```

**Не перезапускайте все брокеры ради одного шкафа.** `mosquitto.service` действует на все
девять сразу, а пересоздание контейнеров (в том числе при загрузке стенда) уничтожает
retained-состояние (см. ниже). Точечный `docker restart` выбранного брокера состояние сохраняет,
а `docker-compose up -d mosquitto_3` — нет: он пересоздаёт контейнер.

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
  образ `eclipse-mosquitto:latest`, порт `1883 + индекс − 1`, монтирования `mosquitto.conf`
  и `passwd`, сеть `sw_net`) и запускает его.
- При переводе шкафа в storage-only соответствующий сервис убирается из compose
  (`removeMosquittoBrokerFromCompose`), но данные брокера и его запись в compose должны
  оставаться восстановимыми: удалять брокер только потому, что бэкенд сейчас остановлен,
  нельзя.
- Правка `docker-compose.yml` пультом — это правка файла в репозитории: изменения нужно
  коммитить и доставлять через PR, как и любые другие.

## Retained-состояние и тома

`/mosquitto/data` — data-каталог образа, в нём лежит `mosquitto.db` (retained-сообщения и
подписки). `/mosquitto/log` — тоже каталог образа. **Ни один из них не объявлен в compose
именованным томом**: Docker создаёт для них анонимные тома из-за `VOLUME` в образе.

Проверено на стенде (изолированный тестовый проект, 2026-09-25, docker-compose 1.29.2):

- **`docker restart`** состояние сохраняет — контейнер тот же, анонимный том остаётся при нём;
- **любое пересоздание контейнера** — `docker-compose up -d --force-recreate`, `up -d` после
  правки compose или образа, а также цикл `down` + `up` — анонимный том **не** переносит:
  compose 1.29.2 (версия 1) создаёт новый пустой том, и `/mosquitto/data` оказывается пустым.
  Именно это делают `ExecStop`/`ExecStart` юнита `mosquitto.service`, то есть **retained-состояние
  не переживает перезагрузку стенда**;
- наблюдение на самом стенде: после включения 2026-09-25 у всех девяти брокеров
  `/mosquitto/data` был пуст, а тома получили время создания, совпадающее со стартом системы
  (контейнеры были пересозданы на загрузке).

Практические следствия:

1. Перед `sudo systemctl restart mosquitto`, перед перезагрузкой стенда и перед любым
   `docker-compose up -d` в этом каталоге (если менялись compose или образ) считайте, что
   retained-состояние всех девяти брокеров будет потеряно. Сохраняет состояние только точечный
   `docker restart` одного брокера.
2. Если устройства полагаются на retained-настройки (команды deep sleep, данные дисплея,
   `unit_weight`), после перезагрузки их нужно опубликовать заново.
3. Надёжное решение — объявить в compose именованные тома на `/mosquitto/data` (например,
   `mosquitto_<индекс>_data`) и перенести туда существующие анонимные тома. Это изменение
   инфраструктуры, его нужно делать отдельной задачей с бэкапом, а не «по ходу» правки
   README. Задача заведена в корневом `SW/BACKLOG.md`.
3. Тома с данными брокеров (`docker volume ls`) не удалять: `docker-compose down -v`
   в этом каталоге недопустим.

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

1. **`docker-compose down` (и `systemctl restart mosquitto`) стирает retained-состояние** —
   см. раздел про тома. Точечный `docker restart` безопаснее.
2. **Один общий `mosquitto.conf` и один общий `passwd`** на девять брокеров: правка
   затрагивает все шкафы, перезапуск — тоже.
3. **Права на `passwd`.** Файл должен читаться uid 1883; иначе брокеры не стартуют.
4. **Пароль один на всех** (`client`), и его хеш лежит в git — при ротации нужно обновить
   файл и перезапустить брокеров, а сам файл вынести из репозитория.
5. **Не удалять брокер остановленного шкафа** и не удалять тома данных: они нужны при
   возврате шкафа в работу.
6. **CRLF.** При правке `docker-compose.yml` и `mosquitto.conf` следите за переводами строк
   (`git diff --ignore-cr-at-eol`), иначе PR выглядит как полная перезапись файла.
