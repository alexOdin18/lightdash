# Проверка Lightdash и работа с проектом Jaffle Shop

## Что делает команда `docker compose --env-file ..\.env up -d`

- **docker compose** — управление сервисами из `docker-compose.yml` (lightdash, db, minio, headless-browser).
- **--env-file ..\.env** — взять переменные окружения из файла `.env` в родительском каталоге (хост, пароли, порт, путь к dbt и т.д.). Без этого подставляются значения по умолчанию или пустые строки.
- **up** — создать и запустить контейнеры.
- **-d** — в фоне (detached), чтобы терминал не был занят выводом логов.

В итоге поднимаются: база Postgres для Lightdash (`db`), MinIO (хранилище), headless-browser и сам Lightdash. Порт 8080 на хосте пробрасывается в контейнер Lightdash, поэтому интерфейс доступен по http://localhost:8080.

**Важно:** команду нужно выполнять из каталога `lightdash-official`, чтобы подхватился правильный `docker-compose.yml`. Путь `..\.env` как раз указывает на ваш `.env` в каталоге `lightdash`.

---

## Что уже настроено

- **DBT_PROJECT_DIR** в `.env` — путь к `jaffle_shop_duckdb` (монтируется в контейнер Lightdash).
- **PGPASSWORD** — пароль для локальной БД Lightdash (сервис `db` в docker-compose).
- Данные Jaffle Shop развёрнуты в облачной PostgreSQL (схема `jaffle_shop`, база `lightdash`).

---

## 1. Запуск Lightdash

Из каталога **lightdash-official** (чтобы подхватился `.env` из родителя или явно указать его):

```powershell
cd c:\Users\intor\lightdash\lightdash-official
docker compose --env-file ..\.env up -d
```

Проверка: откройте в браузере **http://localhost:8080**. Должен открыться интерфейс Lightdash.

---

## 2. Первый вход и организация

1. Зарегистрируйтесь или войдите (email + пароль).
2. Создайте организацию, если предложит (например, «My Org»).
3. Перейдите в настройки организации при необходимости.

---

## 3. Подключение проекта Jaffle Shop

1. В левом меню: **Settings** (или иконка шестерёнки) → **Projects** (или **Create project**).
2. Нажмите **Add project** / **Create new project**.
3. Укажите имя проекта, например: **Jaffle Shop**.

### Тип подключения dbt

- Выберите **"dbt project in repository"** или **"Local dbt project"** (в зависимости от формулировок в вашей версии).
- Для Docker: проект уже смонтирован из `DBT_PROJECT_DIR`, Lightdash видит каталог `/usr/app/dbt` (это ваш `jaffle_shop_duckdb`).
- При запросе **профиля dbt** укажите, что используется профиль **postgres** (target: postgres). При необходимости задайте переменные окружения для dbt в настройках проекта (см. ниже).

### Warehouse (обязательно)

Lightdash должен подключаться к той же PostgreSQL, где лежат данные Jaffle Shop:

- **Connection type:** Postgres.
- **Host:** `d0903710ab9547653313d341.twc1.net`
- **Port:** `5432`
- **Database:** `lightdash`
- **User:** `gen_user`
- **Password:** ваш пароль (тот же, что в `WAREHOUSE_PGPASSWORD` в `.env`).
- **Schema:** `jaffle_shop`
- **SSL:** включить, при необходимости указать **Verify full** и путь к CA-сертификату (в UI может быть поле для сертификата или опция «SSL»).

Сохраните настройки warehouse.

---

## 4. Синхронизация и компиляция

1. После сохранения проекта Lightdash должен предложить **синхронизацию** или **компиляцию** dbt (обновить манифест из dbt).
2. Запустите **Sync** / **Compile** / **Update**. Должны подтянуться модели `customers` и `orders` и метрики из `models/schema.yml`.
3. Если попросят переменные окружения для dbt (для профиля postgres), задайте те же, что в `.env`:
   - `WAREHOUSE_PGHOST`, `WAREHOUSE_PGPORT`, `WAREHOUSE_PGDATABASE`, `WAREHOUSE_PGUSER`, `WAREHOUSE_PGPASSWORD`, `WAREHOUSE_PGSCHEMA`
   - при использовании SSL: `DB_SSL_MODE`, `DB_SSL_ROOT_CERT` (в контейнере путь к серту может быть другим — при необходимости прокинуть сертификат в образ или указать путь внутри контейнера).

---

## 5. Проверка работы с Jaffle Shop

1. В левом меню откройте **Explore** (или **Projects** → выберите **Jaffle Shop**).
2. Выберите **Explore** для одной из таблиц: **orders** или **customers**.
3. В панели измерений и метрик должны быть:
   - для **orders:** Total orders, Total revenue, Average order value, Completed orders, Fulfillment rate, order_date, status и т.д.;
   - для **customers:** Unique customers, Total customer value, first_name, last_name, first_order и т.д.
4. Добавьте несколько полей (dimensions/metrics) и нажмите **Run query**. Должна отобразиться таблица или график по данным из вашей PostgreSQL.
5. Сохраните чарт или создайте дашборд — убедитесь, что сохранение и открытие работают.

Если запрос выполняется и данные отображаются — проект Jaffle Shop в Lightdash настроен и работает корректно.

---

## Возможные проблемы

| Проблема | Что проверить |
|----------|----------------|
| **localhost:8080 не открывается** | Контейнер Lightdash может падать. Выполните `docker ps -a` и проверьте статус `lightdash-official-lightdash-1`. Если **Exited (1)** — смотрите логи: `docker logs lightdash-official-lightdash-1`. Часто причина — **password authentication failed for user "postgres"**: пароль в `.env` (PGPASSWORD) не совпадает с тем, с которым была впервые инициализирована БД в контейнере `db`. Решение ниже. |
| **password authentication failed for user "postgres"** | Контейнер БД при первом запуске запомнил пароль. Если вы потом поменяли PGPASSWORD в `.env`, старые данные в volume остались со старым паролем. Нужно пересоздать volume и контейнеры: `cd lightdash-official` → `docker compose --env-file ..\.env down -v` → `docker compose --env-file ..\.env up -d`. После этого БД инициализируется заново с паролем из текущего `.env`. |
| При запуске `docker compose up` ошибка про volume | В `.env` задан **DBT_PROJECT_DIR** (с прямыми слэшами для Windows). |
| Контейнер `db` не стартует | В `.env` задан непустой **PGPASSWORD** (для сервиса Lightdash). |
| Lightdash не видит модели/метрики | Запущена ли **Sync/Compile** после создания проекта; в настройках проекта указан ли правильный dbt-проект и профиль (postgres). |
| Ошибка при выполнении запроса в Explore | Проверьте настройки **Warehouse**: хост, база, схема `jaffle_shop`, пользователь и пароль; при необходимости SSL. |
| Ошибка SSL при подключении к warehouse из контейнера | В UI Lightdash для Postgres может не быть поля для корневого сертификата. Тогда либо отключите проверку сертификата в тестовой среде (если допустимо), либо настройте SSL-параметры в документации Lightdash для self-hosted. |
| **getaddrinfo ENOTFOUND minio** при запросе | Lightdash сохраняет результаты запросов в S3-совместимое хранилище (в Docker — MinIO). Если контейнер MinIO не запущен, хост `minio` не резолвится. Используйте `docker-compose.override.yml` с официальным образом MinIO (см. файл в lightdash-official) и перезапустите: `docker compose --env-file ..\.env up -d`. Если бакет `default` не создаётся автоматически, создайте его в MinIO Console: http://localhost:9001 (логин minioadmin / minioadmin). |
| **permission denied for database "lightdash"** | Пользователь warehouse (например `gen_user`) не имеет прав на базу `lightdash`. Подключитесь к PostgreSQL под суперпользователем или владельцем БД и выдайте права (см. ниже). |

---

## Права на базу «lightdash» (warehouse)

Если при выполнении запроса в Explore появляется **permission denied for database "lightdash"**, значит роль, под которой Lightdash подключается к warehouse (у вас `gen_user`), не имеет нужных прав в PostgreSQL.

### Что сделать

Подключитесь к серверу PostgreSQL **под суперпользователем или владельцем базы** (например, через pgAdmin, DBeaver или `psql` к базе `postgres` или к `lightdash` под владельцем). Затем выполните:

```sql
-- Разрешить подключение к базе lightdash
GRANT CONNECT ON DATABASE lightdash TO gen_user;

-- Разрешить использование схемы jaffle_shop
GRANT USAGE ON SCHEMA jaffle_shop TO gen_user;

-- Разрешить SELECT по всем таблицам в схеме (уже существующим и будущим)
GRANT SELECT ON ALL TABLES IN SCHEMA jaffle_shop TO gen_user;
ALTER DEFAULT PRIVILEGES IN SCHEMA jaffle_shop GRANT SELECT ON TABLES TO gen_user;
```

Если таблицы создаёт другой пользователь (например, при `dbt run` под другим аккаунтом), то `ALTER DEFAULT PRIVILEGES` нужно выполнить от имени этого пользователя, либо после каждого `dbt run` повторять `GRANT SELECT ON ALL TABLES IN SCHEMA jaffle_shop TO gen_user;`.

После выполнения этих команд снова запустите запрос в Lightdash (Explore → Run query).

---

## Краткий чеклист

- [ ] В `.env`: заданы `PGPASSWORD` и `DBT_PROJECT_DIR`.
- [ ] Выполнено: `docker compose --env-file ..\.env up -d` из `lightdash-official`.
- [ ] Открыт http://localhost:8080, выполнен вход.
- [ ] Создан проект с именем Jaffle Shop, выбран dbt-проект (из смонтированного каталога).
- [ ] Настроен Warehouse: Postgres с параметрами облачной БД и схемой `jaffle_shop`.
- [ ] Выполнена синхронизация/компиляция проекта.
- [ ] В Explore выбран orders или customers, выполнен запрос — данные отображаются.
