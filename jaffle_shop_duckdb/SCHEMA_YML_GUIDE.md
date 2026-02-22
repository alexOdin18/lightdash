# Руководство по заполнению schema.yml в dbt (с учётом Lightdash)

Цель: правильно заполнять YAML-файлы моделей, использовать возможности dbt, соблюдать data governance и DRY, по возможности автоматизировать.

---

## 1. Основные правила и атрибуты

### 1.1 Структура файла

- В начале файла обязательно: `version: 2`.
- Модели описываются в блоке `models:` как список элементов с полями `name`, `description`, `config`, `columns`.

```yaml
version: 2

models:
  - name: my_model
    description: "Краткое описание таблицы и её назначения"
    config:
      meta:
        # настройки уровня таблицы (Lightdash, ключи и т.д.)
    columns:
      - name: column_name
        description: "Описание колонки"
        tests: []
        config: {}
        meta: {}
```

### 1.2 Уровень модели (таблицы)

| Атрибут | Где | Назначение |
|--------|-----|------------|
| `name` | модель | Имя модели (должно совпадать с именем `.sql` файла). |
| `description` | модель | Описание таблицы: что в ней хранится, для кого и для каких сценариев. |
| `config.meta` | модель | Метаданные для Lightdash и других инструментов (см. раздел 3). |

**dbt-тесты на уровне модели** задаются через `config` (например `config: { severity: warn }`), но чаще тесты пишутся на колонках.

### 1.3 Уровень колонки

| Атрибут | Обязательность | Назначение |
|---------|----------------|------------|
| `name` | да | Имя колонки в модели. |
| `description` | рекомендуется | Описание: бизнес-смысл, единицы измерения, PII/чувствительность при необходимости. |
| `tests` | по необходимости | `unique`, `not_null`, `accepted_values`, `relationships` и др. |
| `config` | по необходимости | Конфиг dbt (в т.ч. `meta` для Lightdash — метрики, dimension и т.д.). |
| `meta` | по необходимости | Доп. метаданные; в Lightdash часто используется `meta.dimension` для типа поля. |

**Рекомендации:**

- Для каждой колонки указывать `description` — это основа документации и data governance.
- Повторяющиеся длинные тексты выносить в `docs.md` и подключать через `{{ doc("block_name") }}` (см. раздел 2).
- PII и чувствительные поля явно помечать в описании (например: "Customer's first name. PII.").

### 1.4 Тесты (data quality)

Типичные тесты в `schema.yml`:

```yaml
tests:
  - unique
  - not_null
  - accepted_values:
      values: ['placed', 'shipped', 'completed']
  - relationships:
      to: ref('customers')
      field: customer_id
```

- **unique / not_null** — для ключей и важных полей.
- **accepted_values** — для enum-подобных полей (статусы, типы).
- **relationships** — для внешних ключей (целостность и граф зависимостей dbt).

Тесты лучше описывать в том же файле, где описание модели (один источник правды).

---

## 2. Документация, переиспользование (DRY) и автоматизация

### 2.1 Один источник правды и DRY

- **Один YAML на модель или логическую группу** — не дублировать описание одной и той же модели в нескольких файлах.
- **Длинные описания и справочники** — в `docs.md` (или в отдельном `.md`), в schema — только ссылка:

```yaml
# В schema.yml
- name: status
  description: '{{ doc("orders_status") }}'
```

В `docs.md`:

```markdown
{% docs orders_status %}
Orders can be one of the following statuses:
| status    | description |
|-----------|-------------|
| placed    | ...         |
| completed | ...         |
{% enddocs %}
```

- **Общие списки значений** (статусы, коды) — один раз в `docs`, переиспользование в нескольких моделях.

### 2.2 Где можно автоматизировать

1. **Генерация заготовок YAML по схеме БД**
   - Пакет **dbt-codegen**: макросы `generate_model_yaml()`, `generate_base_model_yaml()` и др. Генерируют блоки `models`/`columns` по уже материализованным таблицам.
   - Установка: в `packages.yml` добавить `dbt-labs/dbt-codegen`, затем `dbt deps`. После `dbt run` можно вызывать макросы и вставлять результат в `schema.yml`.
   - Рекомендуемый поток: сгенерировать каркас → вручную добавить описания, тесты и Lightdash-метрики.

2. **Проверка и линтинг**
   - **dbt:** `dbt compile` — проверка синтаксиса и ссылок.
   - **Lightdash:** `lightdash lint` — проверка семантического слоя (метрики, dimensions, joins). Запускать после изменений в `schema.yml`.

3. **AI / Copilot**
   - Документация Lightdash рекомендует давать AI доступ к схеме хранилища и к [спецификации Lightdash YAML](https://raw.githubusercontent.com/lightdash/lightdash/refs/heads/main/packages/common/src/schemas/json/model-as-code-1.0.json), чтобы генерировать описания колонок и заготовки метрик/дименшенов.
   - После автогенерации — обязательно проверять через `dbt compile` и `lightdash lint`.

4. **Структура папок**
   - Staging-модели можно описывать в `models/staging/schema.yml` (минимально: колонки + тесты).
   - Маршруты (marts) — в `models/schema.yml` с полной документацией и Lightdash (описание, метрики, joins). Так проще поддерживать и не дублировать.

### 2.3 Data governance в schema.yml

- **Описание** — кто, зачем и как может использовать таблицу/колонку.
- **Тесты** — контракт качества (уникальность, допустимые значения, связи).
- **PII/чувствительность** — явно в `description`; при необходимости можно ввести единый префикс или тег в `meta` для последующей фильтрации в каталоге.
- **Владелец/контакт** — при желании можно добавить в `meta` (например `meta: { owner: "analytics-team" }`) и использовать в каталоге данных.

---

## 3. Оформление метрик и таблиц для Lightdash

Lightdash читает семантический слой из **`config.meta`** (и при необходимости из `meta`) в тех же `schema.yml`. Важно: в **dbt 1.10+** метаданные Lightdash задаются в основном через **`config.meta`**, а не только через `meta`.

### 3.1 Что нужно, чтобы модель стала таблицей в Lightdash

- Модель должна быть объявлена в `schema.yml` и иметь **хотя бы одну описанную колонку** (с `name` и желательно `description`). Тогда таблица появится в Lightdash.

### 3.2 Конфигурация таблицы (уровень модели)

В `config.meta` у модели:

| Свойство | Описание |
|----------|----------|
| `primary_key` | Поле — первичный ключ (для joins и отображения). |
| `default_time_dimension` | Поле и интервал по умолчанию для временных срезов: `field: order_date`, `interval: MONTH`. |
| `joins` | Список join'ов к другим моделям (см. ниже). |
| `label` | Название таблицы в UI. |
| `group_label` | Группа таблиц в сайдбаре (например "Sales"). |
| `order_fields_by` | `label` (по умолчанию) или `index` — порядок полей в сайдбаре. |
| `sql_filter` | Постоянный фильтр (например row-level security). |

**Пример joins:**

```yaml
config:
  meta:
    primary_key: order_id
    default_time_dimension:
      field: order_date
      interval: MONTH
    joins:
      - join: customers
        sql_on: ${orders.customer_id} = ${customers.customer_id}
        relationship: many-to-one
        label: Customer
```

- `relationship`: `one-to-many`, `many-to-one` или `one-to-one`.
- В `sql_on` используйте синтаксис `${model_name.column}`.

### 3.3 Dimensions (измерения)

Чтобы колонка была измерением в Lightdash, у колонки задаётся `meta.dimension` (и при необходимости дублируется в `config.meta.dimension` для переопределения):

```yaml
- name: order_date
  description: Date (UTC) that the order was placed
  config:
    meta:
      dimension:
        type: date
        time_intervals: ["DAY", "WEEK", "MONTH", "YEAR", "QUARTER_NAME"]
  meta:
    dimension:
      type: date
```

Типы dimension: `string`, `number`, `date`, `boolean`, `timestamp` и др. Для дат полезно указывать `time_intervals`.

### 3.4 Метрики (уровень колонки)

Метрики для Lightdash задаются в **`config.meta.metrics`** у колонки (dbt 1.10+):

```yaml
- name: amount
  description: Total amount (AUD) of the order
  config:
    meta:
      metrics:
        total_revenue:
          type: sum
          format: usd
          round: 2
          label: Total revenue
        average_order_value:
          type: average
          format: usd
          round: 2
          label: Average order value
        completed_revenue:
          type: sum
          format: usd
          round: 2
          label: Revenue (completed orders)
          filters:
            - is_completed: "true"
  meta:
    dimension:
      type: number
```

**Важно:**

- Метрики привязаны к колонке, по которой идёт агрегация (например `amount` → sum, average).
- Одна колонка может иметь несколько метрик (sum, average, count и т.д.).

### 3.5 Типы метрик Lightdash (кратко)

| Тип | Категория | Описание |
|-----|-----------|----------|
| `count` | aggregate | Количество строк (COUNT). |
| `count_distinct` | aggregate | Уникальные значения (COUNT DISTINCT). |
| `sum` | aggregate | Сумма. |
| `average` | aggregate | Среднее. |
| `min`, `max` | aggregate | Минимум/максимум. |
| `median`, `percentile` | aggregate | Медиана, перцентиль. |
| `number` | non-aggregate | Выражение по другим метрикам (например `${sum_revenue} / ${count_orders}`). |
| `boolean` | non-aggregate | Условие по метрикам (true/false). |
| `percent_of_total`, `running_total` | post-calculation | Доля от итога, накопительный итог (experimental). |

- **Aggregate-метрики** ссылаются только на dimensions (или на SQL по колонкам).
- **Non-aggregate** (`number`, `boolean`) ссылаются только на другие метрики в `sql`.

### 3.6 Свойства метрики

| Свойство | Описание |
|----------|----------|
| `type` | Обязательно. Один из типов выше. |
| `label` | Название в UI. |
| `description` | Описание метрики. |
| `format` | `usd`, `percent`, `eur` или кастомный (например `#,##0.00`). |
| `round` | Округление (число знаков). |
| `sql` | Кастомное SQL для метрики (например для условной агрегации или преобразования). |
| `filters` | Список фильтров по dimension (например `is_completed: "true"`). |
| `hidden` | `true` — скрыть из UI. |
| `groups` | Группировка в сайдбаре. |
| `show_underlying_values` | Какие dimensions показывать в "View underlying data". |

**Пример метрики с кастомным SQL:**

```yaml
fulfillment_rate:
  type: average
  format: percent
  round: 1
  sql: "CASE WHEN ${is_completed} THEN 1 ELSE 0 END"
  label: Fulfillment rate
```

### 3.7 Метрики на уровне модели

Если метрика зависит от нескольких колонок или других метрик, её задают в `config.meta.metrics` у **модели** (не у колонки):

```yaml
models:
  - name: orders_model
    config:
      meta:
        metrics:
          revenue_per_user:
            type: number
            sql: ${sum_revenue} / NULLIF(${distinct_user_ids}, 0)
```

Здесь `sum_revenue` и `distinct_user_ids` должны быть определены как метрики в колонках этой модели.

### 3.8 Чеклист перед выкатом изменений

1. `dbt compile` — без ошибок.
2. `dbt test` — тесты проходят.
3. `lightdash lint` — семантический слой корректен.
4. В Lightdash после деплоя: обновить проект (Refresh dbt) и проверить таблицы/метрики в UI.

---

## Полезные ссылки

- [Lightdash Metrics Reference](https://docs.lightdash.com/references/metrics/)
- [Lightdash Tables Reference](https://docs.lightdash.com/references/tables/)
- [Lightdash YAML (standalone)](https://docs.lightdash.com/guides/lightdash-yaml) — синтаксис совместим с dbt meta.
- [dbt-codegen](https://github.com/dbt-labs/dbt-codegen) — генерация YAML по схеме.
- [dbt docs](https://docs.getdbt.com/reference/dbt-jinja-functions/doc) — макрос `doc()` для переиспользования описаний.
