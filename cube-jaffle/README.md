# Cube.js — семантический слой для Jaffle Shop (DuckDB)

Проект подключает [Cube.js](https://cube.dev/) к данным, собранным dbt в `jaffle_shop_duckdb`, и даёт единый семантический слой (модели, метрики, измерения) поверх DuckDB.

Подход как в статье [Building A Meaningful Semantic Layer](https://performancede.substack.com/p/building-a-meaningful-semantic-layer): **запуск через Docker, Node.js не нужен.**

## Что уже сделано

- **Подключение к DuckDB**: используется файл `jaffle_shop_duckdb/jaffle_shop.duckdb` (создаётся после `dbt build`).
- **Кубы (семантический слой)**:
  - **customers** — клиенты и метрики (количество заказов, сумма заказов).
  - **orders** — заказы, суммы по способам оплаты, связь с `customers`.

## Требования

1. **Docker** (и Docker Compose) — достаточно для запуска Cube; Node.js не нужен.
2. **Данные в DuckDB** — один раз выполните в каталоге `jaffle_shop_duckdb`:
   ```bash
   cd jaffle_shop_duckdb
   dbt deps
   dbt seed
   dbt build
   ```
   После этого появится файл `jaffle_shop.duckdb`.

## Запуск (рабочий прототип)

Из корня репозитория или из папки `cube-jaffle`:

```bash
cd cube-jaffle
docker compose up
```

- **Cube Playground (веб-интерфейс)**: откройте в браузере **http://localhost:4000**
- Там можно выбирать измерения и меры (customers, orders), менять группировки и смотреть сгенерированный SQL — это и есть «рабочий прототип» семантической модели.

(Опционально порт **15432** — Postgres-эндпоинт для подключения BI-инструментов: Tableau, Power BI и т.д.)

## Проверка работы

1. В Playground (http://localhost:4000) выберите, например, меры `orders.count`, `orders.total_amount` и измерение `orders.order_date` (по месяцу/году) и нажмите Run.
2. Или вызовите API в браузере:
   ```
   http://localhost:4000/cubejs-api/v1/load?query={"measures":["orders.count","orders.total_amount"]}
   ```

## Конфигурация (Docker)

- **docker-compose.yml** — образ `cubejs/cube`, порты 4000 и 15432, монтирование папки `cube-jaffle` в `/cube/conf` и папки `jaffle_shop_duckdb` в `/data`; внутри контейнера путь к БД: `/data/jaffle_shop.duckdb`.
- Файл **.env** используется при локальном запуске через Node.js; при Docker переменные заданы в `docker-compose.yml`.

## Структура модели (семантический слой)

- `model/cubes/customers.yml` — куб клиентов (dimensions + measures).
- `model/cubes/orders.yml` — куб заказов и join к `customers`.

Дальше можно добавлять новые кубы, измерения и метрики в этих файлах или создавать новые файлы в `model/cubes/`.

## Запуск без Docker (опционально)

Если хотите запускать Cube без Docker, понадобится **Node.js**. Тогда:

```bash
cd cube-jaffle
npm install
npm run dev
```

Тот же Playground будет на http://localhost:4000.

## Полезные ссылки

- [Cube Docs — DuckDB](https://cube.dev/docs/product/configuration/data-sources/duckdb)
- [Cube Data Model](https://cube.dev/docs/product/data-modeling/overview)
