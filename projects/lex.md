# Lex

AI-SaaS для автоматизации юридической рутины: интейк клиентов, поиск судебной практики, генерация документов, подача через ГАС Правосудие, мониторинг хода дел.

## Что делает

- **Интейк** — запись разговора с клиентом, транскрипция, квалификация дела
- **RAG по судебной практике** — гибридный поиск (BM25 + embeddings) по парсингу sudact.ru
- **Генерация процессуальных документов** — иски, отзывы, ходатайства, с версионированием и редактированием в TipTap
- **Подача документов** — браузерный агент через ГАС Правосудие
- **Мониторинг дел** — оркестратор фоновых задач, уведомления о статусах

## Архитектура

```
apps/
├── web/         Next.js 15 App Router — UI клиента и юриста
└── api/         Hono backend (Node.js)

packages/
├── db/          Drizzle ORM схемы и миграции
├── ui/          Дизайн-токены, shared shadcn-компоненты
├── shared/      zod-схемы, типы, утилиты
└── config/      shared TS / ESLint / Prettier

docker/
└── postgres/    Кастомный образ Postgres + pgvector + pg_bm25
```

## Стек

| Слой | Технология |
|---|---|
| Frontend | Next.js 15, TypeScript, Tailwind, shadcn/ui, Zustand, React Query, TipTap |
| Backend | Hono, Better Auth, Drizzle ORM, BullMQ, Zod |
| База данных | PostgreSQL 16 + pgvector + pg_bm25 (Paradedb) |
| Storage | MinIO (S3-compatible) |
| Очереди и кэш | Redis |
| LLM | Claude Sonnet через прокси-шлюз |
| Транскрипция | Whisper large-v3 |
| Browser automation | Browserbase + Playwright |
| Build | pnpm-monorepo, Turbo |

## Ключевые решения

- **Multi-tenancy с первого дня** — каждый юрист/контора изолирован на уровне БД, без `WHERE tenant_id = ?` в коде (через RLS).
- **Гибридный поиск** — pgvector даёт семантическую близость, pg_bm25 — точные совпадения терминов; результаты сливаются с весом.
- **Browser-агент в облаке** — Browserbase даёт изолированные сессии Playwright без инфры на нашей стороне.
- **Sprint-driven roadmap** — 6 спринтов от скелета до интеграции с ГАС Правосудие.

## Статус

Private repo. По доступу — пишите.
