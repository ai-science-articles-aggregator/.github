# Интеллектуальный агрегатор научных статей

Инструмент для исследователей, студентов и аналитиков, который меняет подход к обзору научной литературы. Загрузите несколько ключевых слов по вашей теме — система найдёт релевантные статьи, сопоставит их содержание и сгенерирует связный итоговый текст.

## 👥 Команда

| Участник                                                    | @telegram     |
| ----------------------------------------------------------- | ------------- |
| [Мовсумов Денис Асифович](https://github.com/DMovsumov)     | @Denis_movsum |
| [Коноплев Роман Алексеевич](https://github.com/Hollenhaunf) | @hollenhanf   |
| [Крылов Александр Вячеславович](https://github.com/Kaderah) | @ooooofsa     |
| [Абакумов Юрий Максимович](https://github.com/fluloeo)      | @fluloeo      |

**Научный руководитель:** [Бурлова Альбина Сергеевна](https://github.com/AlbinaBurlova), старший преподаватель ФКН

## 📊 Как это работает

1. Пользователь добавляет статьи в ноутбук (PDF, URL, текст)
2. **retrieval**-сервис ищет релевантные статьи по запросу через гибридный поиск (e5-large-v2 + BM25 + RRF → reranker)
3. **agent**-сервис читает статьи и генерирует стриминговый ответ через LLM
4. Ответ в реальном времени возвращается в браузер через SSE.

## Ключевые возможности

* 📖 Глубокий контекстный анализ: Понимает смысл статей
* 🧩 Интеллектуальное слияние: Объединяет статьи логически, создавая связный текст
* 📃 Многоформатный вход: Работает с текстовыми файлами (.txt), PDF-документами, url и сырым текстом.
* 🎯 Адаптивное суммаризирование: Генерирует итог заданной длины, фокусируясь на самом важном.

## Репозитории

| Репозиторий | Стек | Описание |
|---|---|---|
| [margin](https://github.com/ai-science-articles-aggregator/margin) | SvelteKit, TypeScript | Фронтенд, SSR/BFF |
| [core-engine](https://github.com/ai-science-articles-aggregator/core-engine) | FastAPI, Python, SQLAlchemy | REST API, gRPC-клиент, бизнес-логика |
| [ml_cores](https://github.com/ai-science-articles-aggregator/ml_cores) | Python, gRPC, LangGraph | ML-сервисы: retrieval и agent |
| [citadel](https://github.com/ai-science-articles-aggregator/citadel) | Docker Compose, Nginx, GitHub Actions | Инфраструктура и CI/CD |
| [proto_contracts](https://github.com/ai-science-articles-aggregator/proto_contracts) | Protobuf | gRPC-контракты |
| [spider](https://github.com/ai-science-articles-aggregator/spider-collector) | Grobid, Airflow, PostgreSQL | Парсер статей |

## Архитектура

```
Browser
  │  HTTP · SSE
  ▼
Nginx (reverse proxy)
  ├── /          → margin (SvelteKit :3000)
  └── /api/      → core-engine (FastAPI :8000)
                       │
              ┌────────┴────────┐
              │                 │
        gRPC :50051       gRPC :50052
              │                 │
          retrieval           agent
      e5-large-v2+BM25    LangGraph+LLM
        RRF → reranker    vLLM/OpenRouter
              │                 │
              └────────┬────────┘
                       ▼
                  arxivdb (PostgreSQL + pgvector)
                  — статьи, чанки, эмбеддинги —

core-engine также читает arxivdb read-only (метаданные статей)
и пишет в отдельную appdb (PostgreSQL 17) — пользователи, ноутбуки, заметки
```

## Быстрый старт (продакшн)

```bash
git clone https://github.com/ai-science-articles-aggregator/citadel
cd citadel
cp .env.example .env
# заполнить .env
docker compose up -d
```

> ML-сервисы (retrieval и agent) разворачиваются отдельно из [ml_cores](https://github.com/ai-science-articles-aggregator/ml_cores) и требуют GPU или ключ OpenRouter (Реализована возможность запуска как локально с использованием ресурсов локальной машины, так и возможность использовать облачные вычисления).

## Локальная разработка

Каждый репозиторий запускается независимо. Подробные инструкции — в README каждого сервиса:

- [core-engine/README.md](https://github.com/ai-science-articles-aggregator/core-engine#readme) — FastAPI, Python 3.13, uv, Alembic
- [margin/README.md](https://github.com/ai-science-articles-aggregator/margin#readme) — SvelteKit, pnpm
- [ml_cores/README.md](https://github.com/ai-science-articles-aggregator/ml_cores#readme) — gRPC-сервисы, Docker, GPU
- [spider/README.md](https://github.com/ai-science-articles-aggregator/spider-collector#readme) – парсер статей, airflow, postgresql

## CI/CD

```
push → main  ▸  GitHub Actions  ▸  Build & Push → GHCR  ▸  dispatch deploy  ▸  docker compose up
```

При пуше в `main` любого репозитория GitHub Actions собирает Docker-образ, пушит в GHCR и отправляет `repository_dispatch` в `citadel`, который обновляет контейнеры на сервере.
