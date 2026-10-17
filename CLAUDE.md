# Контекст проекта: Java-бэкенд извлечения атрибутов товаров (кейс М.Видео/Магнит)

Этот файл — память для Claude Code между сессиями. Источники: `backend-tz.md` (ТЗ Java-бэкенда, главный документ)
и `magnit-attributes-plan.pdf` (исходный план кейса). При расхождении **ТЗ важнее плана**.

## Где живёт настоящий код

- Сдача: только ветка `main` репозитория Gitea `tochka-sborki-2026-team-006/project`, код в папке `backend/`.
- Этот GitHub-репозиторий (`mtlmv/text`) — не репозиторий сдачи. Код отсюда нужно перенести в Gitea.
- Автопроверка читает `main` ежедневно 06:00–10:00, push в это время закрыт → пушить до 06:00.
- Данные (`data/`), `artifacts/category_schema.json`, `artifacts/value_dicts.json` есть только на ВМ команды.
  В git не коммитить данные и ключи. ВМ: 4 vCPU, 8 ГБ RAM, Java нет (ставить openjdk-21 + maven).
- Бастион ВМ: не больше 15 новых SSH-подключений в минуту.

## Дедлайны (что должно быть в `main`)

| Дата | Что | Готово, если |
|---|---|---|
| 18.10 | Каркас: Maven, Java 21, DTO, `Layer`, `Pipeline`, CLI `--mode=predict`, все атрибуты `missing` | валидный `predictions_rules_test.jsonl` по всем тестовым карточкам |
| 19.10 | Слой A + нормализация + юнит-тесты | Python-оценка показывает P/R/F1 слоя A |
| 20.10 | Слой B (бренд, модель, числа, цвет) | бренд F1 ≥ 0,9 на тесте; абляция A → A+B |
| 21.10 | LLM для пустых required, статусы, `POST /extract` | каскад A+B+LLM; падение LLM не валит прогон |
| 22.10 | Dockerfile, порт 8000, прогон по 1,18 млн карточек, README | `data/predictions_all.jsonl` готов |
| 23.10 17:00 | Технический чекпоинт с отсевом | демо + код в `main` |
| 24–28.10 | Улучшения правил, очередь проверки (плюс) | |
| 29.10 00:01 | Стоп-код | |

Приоритет при нехватке времени: CLI + слой A → слой B (бренд/модель/числа) → LLM → API → Docker.

## Пайплайн

```
карточка {sku_id, category_id?, title, description}
 0. категория: из входа (cards.jsonl уже содержит category_id, в т.ч. предсказанный)
 1. схема: required + прочие атрибуты из category_schema.json
 2. Слой A (source=seller): блок «Характеристики» в описании (29% карточек)
    HTML <br>/<p>/<li> → \n; строки «• Ключ: значение» / «- Ключ: значение» / «Ключ: значение»
    ключ → атрибут: точное после нормализации → словарь синонимов → Jaro-Winkler ≥ 0,92
 3. Слой B (source=rules): бренд (vendors.json, самое длинное совпадение в title), модель, память/RAM
    («8/256 ГБ» → RAM 8, память 256), диагональ, батарея, число+единица, цвет, да/нет-фичи
 4. Нормализация: единицы схемы, ё→е, регистр, синонимы → каноническое написание из value_dicts.json
    (самый частый вариант). Нет в справочнике → оставить, уверенность ×0,8
 5. LLM: только пустые required, ОДИН POST /extract на карточку; enum если вариантов ≤ 30, иначе top-30 values;
    таймаут 60 с, 1 повтор; ошибка/missing → атрибут missing, прогон не падает; --llm-limit N (по умолчанию 0);
    шлюз: 2 параллельных запроса и 100 ₽/сутки на команду (~3 тыс. карточек)
 6. Уверенность: seller 0,9; rules 0,85 (бренд/память/диагональ/батарея), 0,7 прочее; llm — из ответа;
    слои согласны +0,05. Статус: accept ≥ 0,8, review 0,6–0,8, drop < 0,6, не нашли — missing
```

## Выходной контракт (одна строка JSONL = одна карточка)

```json
{"sku_id":"30055584","category_id":"761","attributes":[
 {"name":"Бренд","value":"Samsung","confidence":0.92,"source":"rules","status":"accept",
  "evidence":{"text":"Samsung","start":9,"end":16,"field":"title"}},
 {"name":"Цвет","value":null,"confidence":0.0,"source":null,"status":"missing","evidence":null}]}
```

- `name` — ровно как в `category_schema.json`. В списке все required (пустые — missing) + найденные прочие.
- `source`: `seller|rules|llm`, `status`: `accept|review|drop|missing` (нижний регистр, не как в PDF-плане).
- Метрики в Java не считаем — это `python -m scripts.eval_predictions <файлы>`.

## CLI и API

```
java -jar backend.jar --mode=predict --input data/cards_test.jsonl --output artifacts/predictions_rules_test.jsonl --layers=A,B
java -jar backend.jar --mode=predict --input data/cards_test.jsonl --output artifacts/predictions_cascade_test.jsonl --layers=A,B,LLM --llm-limit=300
```

API (Spring Boot, Jackson snake_case, CORS открыт, springdoc, порт 8000):
`POST /extract` (выход = строка контракта + `category_name`, `usage{llm_calls,cost_rub,latency_ms}`, `required` у атрибута),
`GET /schema/{category_id}`, `GET /health`. Статика фронта: `src/main/resources/static`.
LLM-сервис сейчас на `localhost:8000`, должен переехать на 8001 (договориться с Абдулом).
БД и очередь проверки — только после чекпоинта.

## Структура `backend/` (согласована)

```
backend/
├── pom.xml  Dockerfile  README.md
└── src/main/java/<base>/
    ├── BackendApplication.java      # --mode=predict → CLI без веб-сервера, иначе Spring Web
    ├── cli/                         # PredictCommand, CliOptions (--input/--output/--layers/--llm-limit)
    ├── config/                      # пути к артефактам, Jackson snake_case, CORS, загрузка JSON-артефактов
    ├── model/                       # record-DTO: Card, Prediction, AttrResult, Evidence, Source, Status
    │   └── schema/                  # CategorySchema, AttributeSpec (type, unit, enum/values, required)
    ├── artifacts/                   # CategorySchemaRepository, ValueDictionary, VendorDictionary, Thresholds
    ├── text/                        # TextCleaner (HTML→\n, ё→е), Span — позиции считаются по исходному тексту
    ├── pipeline/                    # Layer, ExtractionContext, Candidate, Pipeline, Merger, ConfidenceScorer, StatusAssigner
    ├── layer/seller/                # слой A: SpecsBlockParser, KeyMatcher, SellerSpecsLayer
    ├── layer/rules/                 # слой B: Brand/Model/Memory/Diagonal/Battery/UnitQuantity/Color/BooleanFeature
    ├── layer/llm/                   # LlmClient, LlmRequest/Response, LlmBudget, EvidenceGuard, LlmLayer
    ├── normalize/                   # NumberParser, UnitConverter, StringCanonicalizer, Synonyms
    └── api/                         # ExtractController, SchemaController, HealthController, dto/
src/main/resources/  application.yml, synonyms/*.json (наши синонимы ключей и цветов), static/
src/test/java/...    тесты по пакетам normalize/, layer/seller/, layer/rules/, pipeline/
src/test/resources/  маленькие сэмплы карточек и схемы (без реальных данных)
```

## Открытые вопросы

- Нужны из репозитория Gitea: `contracts/README.md`, `artifacts/category_schema.sample.json`, `artifacts/thresholds.json`,
  `frontend-tz.md` (раздел 4 — контракт `POST /extract`).
- Размер `value_dicts.json` и `category_schema.json` (влезут ли в heap на ВМ с 8 ГБ рядом с LLM-сервисом).
- Проверяет ли LLM-сервис цитату (evidence) сам, или это делать в Java (`EvidenceGuard`).
