# Sales Agent

Channel-agnostic AI-агент для продаж. Один «AI-мозг» обслуживает диалоги с клиентами через любой канал (Telegram, Max, TenChat, Instagram, ...) и доводит до созвона / номера телефона / handoff в CRM.

## Что делает

- Принимает входящие сообщения от любых транспортов через единое REST API
- Ведёт диалог с клиентом на основе версионируемого system-prompt и базы знаний
- Использует tool-calling для поиска по KB, фиксации телефонов, бронирования звонков
- При триггере handoff — создаёт сделку в CRM и стреляет HMAC-подписанный webhook в транспорт
- Multi-tenant: один инстанс — N агентов (бизнесов) с разными промптами, KB, CRM-настройками
- UI с полной прозрачностью: токены, стоимость, JSON-трейс каждого хода, tool calls

## Архитектура

```
┌──────────────────┐
│  Транспорт A     │──┐
│  (Telegram)      │  │
├──────────────────┤  │
│  Транспорт B     │──┼─►┌─────────────────┐         ┌──────────────┐
│  (Max)           │  │  │   sales-agent   │────────►│    CRM       │
├──────────────────┤  │  │   (port 3100)   │         │   (handoff)  │
│  Транспорт C     │──┘  │                 │◄────────┤              │
│  (TenChat / ...) │     └────────┬────────┘         └──────────────┘
└──────────────────┘              │
                                  ▼
                    ┌──────────────────────────┐
                    │  SQLite (data/agent.db)  │
                    │  - agents                │
                    │  - prompts (versioned)   │
                    │  - knowledge_base (xlsx) │
                    │  - conversations         │
                    │  - messages              │
                    │  - audit                 │
                    │  - api_keys              │
                    │  - amo_configs, amo_log  │
                    └──────────────────────────┘
```

## Стек

- **Runtime:** Node.js 20, Express
- **БД:** SQLite в WAL-режиме (better-sqlite3)
- **LLM:** через прокси-шлюз с OpenAI-совместимым API
- **Frontend:** vanilla JS SPA, JWT-auth
- **Развёртывание:** Docker Compose, single host

## Ключевые решения

### Stateless transport, stateful agent

Транспорт не хранит никакой диалоговой памяти. Весь контекст диалога живёт в sales-agent (`conversations`, `messages`). Если транспорт упал и поднялся — он просто шлёт следующее сообщение клиента с тем же `conversation_key`, агент продолжает с того же места.

### Multi-tenant с первого дня

В каждой таблице есть `agent_id`. Один инстанс может обслуживать N разных бизнесов одновременно — у каждого свои промпты, KB, API-ключи, CRM-интеграция, статистика, цена за токен.

### Tool-calling loop

LLM на каждом ходу решает: ответить текстом или вызвать инструмент. Доступные инструменты:

| Tool | Что делает |
|------|-----------|
| `search_kb(query)` | Поиск по загруженным xlsx с ранжированием |
| `save_phone(phone, preferred_time?)` | Сигнал транспорту: клиент дал телефон |
| `book_call(date, time, phone?)` | Сигнал: договорились о созвоне |
| `handoff_to_manager(reason, summary)` | Финал: передать менеджеру через CRM |

Когда агент вызывает `handoff_to_manager`, цикл делает ещё одну итерацию с инструкцией «напиши короткое прощание» — это и уходит клиенту как финальный текст. Параллельно: статус conversation → `handoff`, фиксируется `phone`/`reason`/`summary`, стреляет webhook в транспорт, создаётся сделка в CRM.

### База знаний через tool, а не через context-stuffing

Целиком xlsx не инжектится в system prompt. Вместо этого туда идёт короткий **overview** (1–2 строки на лист), а агент сам решает когда нужен детальный поиск через `search_kb(query)`. Экономия: overview ~500 токенов вместо 10k+ содержимого.

### Webhook на handoff с HMAC-подписью

Транспорт при создании API-ключа указывает `webhook_url`. На handoff туда прилетает POST с HMAC-SHA256 подписью:

```
Headers:
  X-Sales-Agent-Signature: sha256=<hex>
  X-Sales-Agent-Event: handoff

Body:
  { "event": "handoff", "conversation_key": ..., "phone": ..., "summary": ... }
```

### CRM как параллельный канал handoff'а

Если в настройках агента есть CRM-конфиг (subdomain + access_token + pipeline_id), на handoff дополнительно создаётся: контакт → сделка в указанной воронке/этапе → заметка с summary. Работает best-effort: ошибки CRM логируются, но не ломают основной flow.

### Что выбрано сознательно «проще»

- **Нет vector DB / embeddings** — xlsx маленькие, хватает keyword search
- **Нет OAuth2 refresh для CRM** — long-lived integration token
- **Нет очередей (Redis/BullMQ)** — handoff делается синхронно in-process с fire-and-forget в CRM
- **Нет TLS из коробки** — ожидается reverse proxy наружу
- **Нет multi-admin** — один admin user в `.env`, расширяется при необходимости

## Интерфейс

Single-page app с вкладками:

- **Дашборд** — сводка за сегодня (диалоги, handoffs, токены, стоимость)
- **Диалоги** — двухколоночный вид (список / переписка)
- **Промпты** — редактор с версионированием, откат на любую версию одним кликом
- **База знаний** — загрузка xlsx, превью парсинга, вкл/выкл листов
- **Debug / JSON** — полная JSON-цепочка ходов для каждого turn (user msg → tool calls → reply)
- **Интеграции** — API-ключи для каналов, webhook URL
- **CRM** — credentials + проверка подключения + лог отправок
- **Настройки** — модель, temperature, max_turns, вкл/выкл агента

## Статус

Private repo. По доступу — пишите в Telegram [@AleCenn](https://t.me/AleCenn).
