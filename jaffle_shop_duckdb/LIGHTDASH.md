# Подключение Jaffle Shop (DuckDB) к Lightdash

В этом проекте настроен семантический слой Lightdash: метрики, измерения и связи между моделями заданы в `models/schema.yml` и `lightdash.config.yml`.

## Что уже сделано

- **Семантический слой** в `models/schema.yml`:
  - Модели `customers` и `orders` с `primary_key`, `joins`, `default_time_dimension`
  - Метрики: Total orders, Completed orders, Total revenue, Average order value, Fulfillment rate, Unique customers, Total customer value и др.
  - Измерения (dimensions) с типами и, где нужно, интервалами времени
- **Колонка `is_completed`** в модели `orders` для фильтрации выполненных заказов
- **Конфиг Lightdash** в `lightdash.config.yml` (категории для Spotlight)

## Ограничение: DuckDB и Lightdash

Lightdash **не поддерживает DuckDB** как warehouse (поддерживаются Postgres, BigQuery, Snowflake, Redshift, Databricks, Trino, ClickHouse, Athena). Запросы к данным из UI Lightdash идут в warehouse, поэтому для полной работы с этим проектом из Lightdash нужен один из вариантов ниже.

## Варианты использования

### 1. Только dbt + DuckDB (без Lightdash UI)

- Запускайте проект как обычно: `dbt build`, `dbt docs generate`, `dbt docs serve`.
- Семантический слой в YAML пригодится для документации и для будущего использования с Postgres или другим warehouse.

### 2. Lightdash с Postgres (рекомендуется для дашбордов)

Чтобы видеть данные и строить дашборды в Lightdash:

1. **Поднять Lightdash и Postgres** (например, через `docker-compose` из репозитория Lightdash).
2. **Подключить dbt-проект на Postgres** с той же структурой метрик:
   - либо скопировать `models/`, `lightdash.config.yml` и т.д. в отдельный dbt-проект с профилем Postgres и загрузить туда данные (например, из DuckDB экспортом или через dbt seed);
   - либо использовать готовый пример [full-jaffle-shop-demo](https://github.com/lightdash/lightdash/tree/main/examples/full-jaffle-shop-demo), где уже есть Postgres и Lightdash.
3. В настройках проекта в Lightdash указать:
   - **Warehouse:** Postgres (с параметрами вашей БД);
   - **dbt project directory:** путь к этому проекту (или к копии с профилем Postgres).

### 3. Docker Compose (Lightdash из репозитория lightdash-official)

В каталоге с `lightdash-official` в `docker-compose.yml` задаётся монтирование dbt-проекта:

```yaml
volumes:
  - '${DBT_PROJECT_DIR}:/usr/app/dbt'
```

Чтобы использовать **этот** проект (jaffle_shop_duckdb) с Lightdash, нужен warehouse = Postgres:

1. Запустите Postgres и Lightdash (как в `lightdash-official`, с переменными окружения для БД).
2. Создайте отдельный dbt-проект под Postgres с теми же моделями и тем же `schema.yml` / `lightdash.config.yml` (или используйте один каталог с двумя профилями и переключайте `target`).
3. Установите `DBT_PROJECT_DIR` на путь к этому dbt-проекту (с профилем Postgres и данными в Postgres).

Если оставить текущий проект с профилем DuckDB и указать его в `DBT_PROJECT_DIR`, Lightdash сможет прочитать манифест и метрики, но **выполнять запросы к данным не сможет**, так как DuckDB не поддерживается как warehouse.

## Проверка семантического слоя локально

```bash
cd jaffle_shop_duckdb
dbt compile
dbt run
```

После этого в `target/manifest.json` и в `target/run_results.json` будет актуальная информация. Lightdash при синхронизации проекта читает манифест и подхватывает метрики и измерения из `meta`.

## Перенос данных и семантического слоя в PostgreSQL

В репозитории уже настроены:

- **Файл `.env`** (в корне `lightdash/`) — переменные для подключения к PostgreSQL. Заполните их (хост, порт, база, пользователь, пароль, схема).
- **Профиль dbt `postgres`** в `profiles.yml` — читает переменные `WAREHOUSE_PGHOST`, `WAREHOUSE_PGPORT`, `WAREHOUSE_PGDATABASE`, `WAREHOUSE_PGUSER`, `WAREHOUSE_PGPASSWORD`, `WAREHOUSE_PGSCHEMA` из окружения.

### Шаги

1. **Заполните `.env`** (в каталоге `lightdash/`):
   - `WAREHOUSE_PGHOST` — хост вашей PostgreSQL (например `localhost` или IP).
   - `WAREHOUSE_PGPORT` — порт (обычно `5432`).
   - `WAREHOUSE_PGDATABASE` — имя базы.
   - `WAREHOUSE_PGUSER` и `WAREHOUSE_PGPASSWORD` — учётные данные.
   - `WAREHOUSE_PGSCHEMA` — схема для моделей (по умолчанию `jaffle_shop`).

2. **Создайте схему в БД** (если её ещё нет):
   ```sql
   CREATE SCHEMA IF NOT EXISTS jaffle_shop;
   ```

3. **Загрузите переменные и запустите dbt** (из каталога `jaffle_shop_duckdb`):
   - Windows PowerShell (заполните пути и значения под себя):
     ```powershell
     cd c:\Users\intor\lightdash\jaffle_shop_duckdb
     Get-Content ..\.env | ForEach-Object { if ($_ -match '^\s*([^#=]+)=(.*)$') { [Environment]::SetEnvironmentVariable($matches[1].Trim(), $matches[2].Trim(), 'Process') } }
     dbt seed --target postgres
     dbt run --target postgres
     dbt test --target postgres
     ```
   - Linux/Mac (bash):
     ```bash
     cd jaffle_shop_duckdb
     set -a; source ../.env; set +a
     dbt seed --target postgres
     dbt run --target postgres
     dbt test --target postgres
     ```

4. **В Lightdash** создайте проект, укажите этот dbt-проект и warehouse **Postgres** с теми же параметрами (хост, база, пользователь, пароль, схема).

После этого семантический слой и данные будут в PostgreSQL, а Lightdash сможет выполнять запросы к ним.

## Итог

- Семантический слой в **jaffle_shop_duckdb** готов для Lightdash.
- Для работы дашбордов и запросов в Lightdash используйте **Postgres**: заполните `.env`, выполните `dbt seed` и `dbt run --target postgres`, затем подключите проект в Lightdash к этой БД.
