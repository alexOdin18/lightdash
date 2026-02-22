# Lightdash + Jaffle Shop

Локальный проект: **Lightdash** (self-hosted BI), **dbt**-проект **Jaffle Shop** с семантическим слоем и данными в **PostgreSQL**.

---

## Состав

| Каталог / файл | Назначение |
|----------------|------------|
| **lightdash-official/** | Исходный репозиторий Lightdash (docker-compose, образы). Запуск из этой папки. |
| **jaffle_shop_duckdb/** | dbt-проект Jaffle Shop: модели customers/orders, семантический слой (метрики и измерения для Lightdash), профили DuckDB и **Postgres**. |
| **reports/** | Ежедневные отчёты (md-файлы с датой в имени). |
| **.env** | Переменные окружения (подключение к Postgres, Lightdash, путь к dbt). Не коммитить. |
| **LIGHTDASH_ПРОВЕРКА.md** | Пошаговая проверка Lightdash и подключения проекта Jaffle Shop. |

---

## Быстрый старт

1. Заполнить **.env** (хост/база/пользователь/пароль Postgres, при необходимости SSL).
2. Развернуть данные в Postgres: из `jaffle_shop_duckdb` выполнить `dbt seed --target postgres`, `dbt run --target postgres` (переменные из .env должны быть в окружении).
3. Запустить Lightdash: из **lightdash-official** выполнить  
   `docker compose --env-file ..\.env up -d`
4. Открыть http://localhost:8080, создать проект, указать dbt (local, target **postgres**, schema **jaffle_shop**) и warehouse Postgres с теми же параметрами из .env.

Подробности — в **LIGHTDASH_ПРОВЕРКА.md** и **jaffle_shop_duckdb/LIGHTDASH.md**.

---

## Ограничения

- Lightdash не поддерживает DuckDB как warehouse; для запросов используется только Postgres (или другой поддерживаемый warehouse).
- Для работы запросов в UI нужен запущенный **MinIO** (S3) и бакет **default** — см. docker-compose.override.yml и инструкции в LIGHTDASH_ПРОВЕРКА.md.
- Подключение к облачной Postgres с самоподписанным сертификатом использует SSL-режим **require** (без проверки сертификата).
