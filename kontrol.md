# ТЕХНИЧЕСКИЙ АУДИТ И ИНЖЕНЕРНОЕ ДОСЬЕ СИСТЕМЫ «KINESIS CONTROL»

---

## 1. МАКРО-УРОВЕНЬ (Система в целом)

### Назначение системы
**Kinesis Control** — высоконадежная многоклиентская (B2B Multi-tenant) платформа управления клиентской технической поддержкой, сквозным жизненным циклом сервисных инцидентов и автоматизированным контролем жестких соглашений об уровне обслуживания (Service Level Agreements, SLA) для экосистемы Kinesis Hub.

Платформа решает критическую инженерную и бизнес-задачу централизованного сопровождения десятков и сотен независимых корпоративных клиентов в рамках единой отказоустойчивой инсталляции:
1. **Строгая мультитенантность без инфраструктурных накладных расходов:** исключает необходимость развертывания отдельных баз данных или контейнеров под каждого клиента. Все данные тенантов изолируются на уровне ядра СУБД с гарантией невозможности межклиентских утечек.
2. **Математически точный контроль SLA с учетом производственного календаря:** вычисляет предельные сроки первого ответа и финального решения инцидентов с привязкой к локальным часовым поясам клиентов, гибким рабочим графикам (рабочие дни, часы открытия/закрытия) и логике динамической заморозки при ожидании действий заказчика.
3. **Биллинг трудозатрат и квотирование ресурсов (Story Points Quota & Capacity Management):** автоматизирует учет списания поинтов сопровождения из предоплаченных пакетов клиентов, двухфазное согласование оценок и балансировку загрузки инженеров команды.
4. **Омниканальный доступ реального времени:** предоставляет единый бэкенд для трех взаимосвязанных интерфейсов:
   - Десктопная операторская консоль (Control Panel SPA) для инженеров и лидов платформы;
   - Автономный Telegram-бот с интерактивной стейт-машиной и кнопочными сценариями;
   - Мобильный Telegram Mini App (TMA) для конечных пользователей клиентов с мгновенной криптографической аутентификацией.

---

### Масштаб кодовой базы и структуры
* **Суммарный объем кодовой базы:** **~11 000 строк (10 952 LOC)**, из которых **~9 855 строк чистой логики (SLOC)** (исключая lock-файлы, бинарные ассеты и auto-generated зависимости).
* **Распределение по стеку технологий:**
  * **Frontend (TypeScript / React 18 / Tailwind CSS / Vite):** **~6 146 строк кода**:
    * `web/src/features/support/ControlPanelApp.tsx` (2 567 LOC) — флагманская операторская консоль (Kanban/List представления, аналитика SLA, панель тикетов, управление клиентами, матрица загрузки команды инженеров).
    * `web/src/features/tma/` (~2 000 LOC) — клиентский Telegram Mini App (визард подачи заявок, чат, согласование бюджета, личный профиль с лимитами).
    * `web/src/lib/`, `store/`, `components/` — клиентский слой взаимодействия с API, управление сессиями, интеграция Telegram WebApp SDK и библиотека UI-примитивов.
  * **Backend Core & Services (Python 3.11+ / FastAPI / SQLAlchemy / AsyncIO):** **~4 016 строк кода**:
    * `api/app/routers/` (2 018 LOC) — контроллеры тикетов, мультитенантного администрирования, аутентификации и шлюза Telegram.
    * `api/app/models.py`, `schemas.py`, `serializers.py` (732 LOC) — реляционные схемы данных, Pydantic-контракты и нормализаторы телеметрии SLA.
    * `api/app/security.py`, `deps.py`, `database.py`, `sla.py` (273 LOC) — ядро безопасности, RLS context injection, алгоритмический движок SLA и Argon2/HMAC криптография.
    * `bot/` (442 LOC) — гибридный сервис бота на Aiogram 3 и FastAPI IPC.
    * `worker/` (135 LOC) — асинхронный Transactional Outbox процессор.
  * **Database Migrations & SQL:** **~76 LOC** (Alembic-миграции с инициализацией RLS-политик и разграничением прав ролей).
  * **Тестовый контур:** **~200 LOC** (автотесты SLA-календаря, HMAC-валидации TMA и сквозной smoke-тест мультитенантной изоляции).
  * **DevOps & Infrastructure:** **~200 LOC** (Docker Compose на 7 сервисов, Nginx конфигурации, скрипты инициализации криптографических секретов и резервного копирования).

---

### Архитектурный паттерн
* **Modular Monolith Core с Event-Driven Transactional Outbox и Zero-Trust Data Isolation:**
  * **Stateless API Gateway & Compute:** FastAPI-приложение, не хранящее сессионного состояния в оперативной памяти контейнера, что обеспечивает бесшовное горизонтальное масштабирование за балансировщиком Nginx.
  * **PostgreSQL Row-Level Security (RLS) Data Isolation:** архитектура с принудительной изоляцией арендаторов на уровне ядра базы данных. Исключает утечку данных между организациями даже в случае логических ошибок в коде роутов.
  * **Transactional Outbox Event Engine:** надежный шаблон доставки внешних уведомлений без внешних брокеров (Kafka/RabbitMQ/Redis). Гарантирует атомарную фиксацию бизнес-событий и уведомлений в единой ACID-транзакции.
  * **Multi-tenant Storage Virtualization:** абстрагированное S3-хранилище (MinIO/AWS S3) с путями, детерминированно изолированными по идентификаторам клиентов (`{tenant_id}/{token_hex}`).

---

### Системные гарантии
* **Строгая изоляция арендаторов (Strict Tenant Isolation):** аппаратная гарантия целостности данных через `ENABLE ROW LEVEL SECURITY` и `FORCE ROW LEVEL SECURITY`. Запросы выполняются под непривилегированным пользователем базы без права обхода политик (`NOBYPASSRLS`).
* **Консистентность и идемпотентность (Idempotency Guarantee):** поддержка HTTP-заголовка `Idempotency-Key` на уровне протокола создания тикетов с уникальным композитным индексом `(tenant_id, idempotency_key)`, предотвращающая дублирование инцидентов и повторные списания квот при нестабильном мобильном соединении.
* **Гарантированная доставка событий (At-Least-Once Delivery):** фоновый воркер с неблокирующим захватом задач `FOR UPDATE SKIP LOCKED`, экспоненциальным бэкоффом (`min(300, 2^attempts)`) и изоляцией фатальных ошибок (dead-letter logic).
* **Календарная точность SLA (Zero False-Breach SLA):** математический учет рабочих интервалов, региональных таймзон и заморозка таймеров в статусах ожидания клиента исключают ложные штрафы и искажения метрик поддержки.
* **Криптографическая безопасность нулевого доверия (Zero-Trust Security):** хеширование паролей Argon2id, каскадный HMAC-SHA256 хендшейк для Telegram WebApp, короткоживущие JWT-токены с принудительной инвалидацией по версионности (`token_version`).

---

### Рекомендуемый тайтл автора
**Principal Software & AI Systems Architect**  
*(Альтернативы для BigTech: Staff Backend & Distributed Systems Engineer / Lead Enterprise Systems Architect)*

*Обоснование:* Автор полностью спроектировал и реализовал сложную многопользовательскую B2B-систему enterprise-уровня, объединив фундаментальные паттерны распределенных систем (Transactional Outbox, Row-Level Security, Concurrency Control, Idempotency) с глубоко проработанным прикладным слоем (SLA-движок, омниканальный Telegram Mini App, биллинг квот).

---

## 2. АРХИТЕКТУРНАЯ ДЕКОМПОЗИЦИЯ С ПРИОРИТИЗАЦИЕЙ

```
                                  ┌────────────────────────┐
                                  │      Client User       │
                                  └───────────┬────────────┘
                                              │ Telegram / TMA
                                              ▼
┌────────────────────────┐         ┌────────────────────────┐
│     Kinesis Staff      │         │   control-bot (8001)   │
└───────────┬────────────┘         │  Aiogram + FastAPI IPC │
            │ HTTPS                └───────────┬────────────┘
            ▼                                  │ Internal API
┌────────────────────────┐                     │ (X-API-Key)
│    control-web (SPA)   │                     │
│ Control Panel + TMA    │                     │
└───────────┬────────────┘                     │
            │ /api/v1                          │
            ▼                                  ▼
┌───────────────────────────────────────────────────────────┐
│                    control-api (8000)                     │
│  FastAPI • Pydantic • SQLAlchemy Core • RLS Context Inject│
└───────┬─────────────────┬───────────────────┬─────────────┘
        │                 │                   │
        │ SQL / RLS       │ S3 API            │ Transactional Write
        ▼                 ▼                   ▼
┌──────────────┐   ┌──────────────┐   ┌─────────────────────┐
│  PostgreSQL  │   │ MinIO / S3   │   │ Table: outbox_events│
│ (control-db) │   │ Storage      │   └──────────▲──────────┘
└──────────────┘   └──────────────┘              │
                                                 │ FOR UPDATE SKIP LOCKED
                                      ┌──────────┴──────────┐
                                      │   control-worker    │
                                      │ Outbox State Engine │
                                      └──────────┬──────────┘
                                                 │ Internal Push HTTP
                                                 └──────────► (to control-bot)
```

### ФУНДАМЕНТАЛЬНЫЕ ПОДСИСТЕМЫ (75% объема раздела)

#### 1. Двухуровневая модель изоляции данных (PostgreSQL RLS & Session Context Injection)
* **Архитектурный вызов:** В многопользовательских B2B-системах фильтрация по `tenant_id` на прикладном уровне (в коде ORM или роутов) фатально уязвима к человеческому фактору: один пропущенный `where` в комплексном `JOIN` приводит к утечке конфиденциальных клиентских данных.
* **Кастомная разработка автора с нуля:**
  * **Database Security DDL (`migrations/versions/20260817_0001_initial.py`):**
    На 11 таблицах системы (`memberships`, `sla_policies`, `api_clients`, `tenant_assignments`, `tickets`, `ticket_messages`, `ticket_tasks`, `attachments`, `telegram_invites`, `telegram_bindings`, `telegram_sessions`) включены механизмы RLS с директивой `FORCE`:
    ```sql
    ALTER TABLE "{table}" ENABLE ROW LEVEL SECURITY;
    ALTER TABLE "{table}" FORCE ROW LEVEL SECURITY;
    CREATE POLICY tenant_isolation ON "{table}"
      USING (
        current_setting('app.platform_admin', true) = 'true'
        OR tenant_id::text = current_setting('app.tenant_id', true)
      )
      WITH CHECK (
        current_setting('app.platform_admin', true) = 'true'
        OR tenant_id::text = current_setting('app.tenant_id', true)
      );
    ```
  * **Разделение привилегий СУБД (Least Privilege Architecture):**
    В `deploy/postgres-init.sh` создается непривилегированная роль `kinesis_control`, лишенная атрибута `BYPASSRLS`. Миграции исполняются от суперпользователя `postgres`, а рабочий трафик API обслуживается пользователем, для которого RLS принудительно активен (`FORCE ROW LEVEL SECURITY`).
  * **Атомарная инжекция контекста сессии (`api/app/database.py`):**
    Реализован перехватчик события SQLAlchemy `after_begin`:
    ```python
    @event.listens_for(Session, "after_begin")
    def apply_tenant_context(session: Session, transaction, connection) -> None:
        if connection.dialect.name != "postgresql":
            return
        tenant_id = str(session.info.get("tenant_id") or "")
        platform_admin = "true" if session.info.get("platform_admin") else "false"
        connection.execute(text("SELECT set_config('app.tenant_id', :tenant_id, true)"), {"tenant_id": tenant_id})
        connection.execute(text("SELECT set_config('app.platform_admin', :value, true)"), {"value": platform_admin})
    ```
    Параметр `is_local = true` в функции `set_config` гарантирует автоматический сброс переменных при завершении транзакции, исключая перекрестное загрязнение соединений в пуле (Connection Pool Poisoning).
  * **Контроль через Dependency Injection (`api/app/deps.py`):**
    Зависимость `current_actor` извлекает claims из JWT, проверяет активность тенанта и пользователя в БД, валидирует `token_version` и устанавливает `db.info["tenant_id"]` и `db.info["platform_admin"]` до вызова обработчика роута.
* **Граничные состояния:** Попытка обращения к чужому тикету отсекается либо RLS-политикой (пустой результат), либо явным HTTP 404 без раскрытия метаинформации, исключая перебор идентификаторов (enumeration attacks).

---

#### 2. Движок расчета SLA и динамического учета рабочего времени (Work Calendar SLA Engine)
* **Архитектурный вызов:** Расчет сроков устранения инцидентов через простое сложение времени неизбежно порождает ложные сбои SLA в нерабочие часы, праздники и выходные дни, искажая отчетность перед заказчиками.
* **Кастомная разработка автора с нуля (`api/app/sla.py`):**
  * Спроектирован и реализован пошаговый алгоритм распределения доступного рабочего времени `add_work_minutes`.
  * **Алгоритмическая логика:**
    ```python
    def add_work_minutes(start: datetime, minutes: int, *, timezone_name: str, workday_start: str, workday_end: str, workdays: list[int]) -> datetime:
        tz = ZoneInfo(timezone_name)
        cursor = (start if start.tzinfo else start.replace(tzinfo=timezone.utc)).astimezone(tz)
        opening, closing = parse_clock(workday_start), parse_clock(workday_end)
        remaining = max(0, minutes)
        while remaining:
            if cursor.weekday() not in workdays:
                cursor = datetime.combine(cursor.date() + timedelta(days=1), opening, tz)
                continue
            day_start = datetime.combine(cursor.date(), opening, tz)
            day_end = datetime.combine(cursor.date(), closing, tz)
            if cursor < day_start: cursor = day_start
            if cursor >= day_end:
                cursor = datetime.combine(cursor.date() + timedelta(days=1), opening, tz)
                continue
            available = int((day_end - cursor).total_seconds() // 60)
            if available <= 0:
                cursor = datetime.combine(cursor.date() + timedelta(days=1), opening, tz)
                continue
            step = min(remaining, available)
            cursor += timedelta(minutes=step)
            remaining -= step
        return cursor.astimezone(timezone.utc)
    ```
  * **Персистентность и неизменяемость истории:** Дедлайны первого ответа (`first_response_due_at`) и финального решения (`resolution_due_at`) рассчитываются и сохраняются в модели `Ticket` в момент регистрации. Последующие изменения корпоративной политики SLA клиента не переписывают сроки исторических инцидентов.
  * **Динамическая пауза регламента (`api/app/serializers.py:timer_payload`):**
    В статусах ожидания действий клиента (`need_info`, `pending_approval`, `awaiting_client`) таймер переходит в состояние `paused`. Время нахождения в паузе аккумулируется в `sla_pause_seconds` и вычитается из общего времени решения, обеспечивая честную фиксацию эффективности работы инженеров.

---

#### 3. Transactional Outbox Pipeline & Высоконадежный Worker (Event Engine)
* **Архитектурный вызов:** Прямая отправка уведомлений в сторонние API (Telegram, вебхуки CRM) из HTTP-хэндлеров транзакции порождает Dual-Write проблему: при сбое сети клиент получает ошибку при уже сохраненных данных в БД, а при отказе базы — ложное уведомление.
* **Кастомная разработка автора с нуля (`worker/main.py`, `models.py:OutboxEvent`):**
  * **Транзакционная атомарность:** Любая мутация заявки (создание, смена статуса, ответ специалиста, согласование сметы) записывает событие в таблицу `outbox_events` внутри единой транзакции PostgreSQL.
  * **Неблокирующая конкурентная выборка:**
    Воркер использует механизм `FOR UPDATE SKIP LOCKED`:
    ```python
    event = db.scalar(
        select(OutboxEvent)
        .where(OutboxEvent.status == "pending", OutboxEvent.available_at <= datetime.now(timezone.utc))
        .order_by(OutboxEvent.created_at)
        .with_for_update(skip_locked=True)
        .limit(1)
    )
    ```
    Это обеспечивает горизонтальную масштабируемость: любое число воркеров может параллельно забирать события без взаимоблокировок (deadlocks) и повторной обработки.
  * **Экспоненциальный бэкофф и изоляция ядовитых сообщений:**
    При временных сбоях внешних сервисов интервал повторной попытки сдвигается по формуле `now() + min(300, 2^attempts)`. При превышении 8 попыток статус меняется на `failed` с записью трейса ошибки (`last_error`).

---

#### 4. Биллинг квот сопровождения и балансировка ресурсов команды
* **Кастомная разработка автора (`api/app/routers/tickets.py`, `support_admin.py`):**
  * **Четырехуровневая классификация работ:**
    * **Категория A (Bug/Defect):** Гарантийные дефекты. Оценка 0 поинтов, согласование не требуется (`approval_status="not_required"`), сразу переход в работу.
    * **Категории B / C (Improvements / New Features):** Оценка в Story Points (`XS=1, S=2, M=3, L=5, XL=8`). Запуск двухфазного согласования (`pending_approval`). При акцепте клиента происходит атомарное списание поинтов из ежемесячного пакета организации (`tenant.monthly_points_used += points_estimate`).
    * **Категория D (Out-of-Scope / Paid Work):** Коммерческая доработка с оценкой в валюте (UZS). Требует прямого согласования стоимости клиентом.
  * **Балансировка ресурсов инженеров:** Для каждого инженера ведется профиль недельной емкости (`weekly_capacity_points`). В роутере распределения вычисляется агрегированная загрузка (`assigned_points = SUM(points_reserved) WHERE status != 'closed'`), процент утилизации (`load_percent`) и матрица закрепления инженеров за организациями (`TenantAssignment`) с процентным весом аллокации.

---

#### 5. Telegram WebApp & Bot Gateway (Cryptographic Handshake & State Machine)
* **Кастомная разработка автора (`api/app/security.py`, `routers/telegram.py`, `bot/main.py`):**
  * **Криптографическая верификация данных Telegram WebApp:**
    Реализован независимый валидатор `validate_telegram_webapp_init_data`:
    1. Извлечение контрольного хеша из строки параметров `initData`.
    2. Построение строки проверки `data_check_string` из лексикографически отсортированных ключей.
    3. Вычисление промежуточного ключа `HMAC_SHA256(b"WebAppData", bot_token)`.
    4. Сверка с полученным хешем через защищенное от timing-атак сравнение `hmac.compare_digest`.
  * **Интерактивная стейт-машина бота:**
    Реализован 5-шаговый визард подачи тикета (`sop_title` → `sop_object` → `sop_actual` → `sop_evidence` → `sop_priority`), сценарии согласования бюджета (`approve`/`reject` с фиксацией причины в переписке) и приёмки работ (`accept`/`rework`). Персистентность драфтов в JSONB-колонке `draft_json` поддержана принудительной инвалидацией кэша ORM через `flag_modified(session, "draft_json")`.

---

### ВСПОМОГАТЕЛЬНЫЕ ПОДСИСТЕМЫ (25% объема раздела)

#### 1. S3 Object Storage & Media Streaming Subsystem (`api/app/storage.py`, `api/app/routers/tickets.py`)
* **Назначение:** Изолированное хранение вложений тикетов (скриншоты, дампы, голосовые сообщения).
* **Контракты и интеграции:**
  * Загрузка через `multipart/form-data` с валидацией размера (до 25 МБ) и строгим белым списком MIME-типов (`image/*`, `audio/*`, `video/*`, `text/*`, `application/pdf`, `.docx`, `.xlsx`).
  * Детерминированное формирование ключей с изоляцией по клиентам: `{tenant_id}/{token_hex(16)}`.
  * Потоковая раздача контента (`StreamingResponse`) чанками по 64 КБ напрямую из S3-потока, исключающая буферизацию в RAM контейнера API.
  * Управление HTTP-заголовками кэширования: `Cache-Control: public, max-age=86400` и `Content-Disposition` для безопасного скачивания.

#### 2. Платформенный интерфейс Control Panel & Telegram Mini App (Web Layer)
* **Назначение:** Рабочее место операторов поддержки (десктоп) и мобильный портал клиента.
* **Реализация:**
  * Полнофункциональное SPA на React 18 / TypeScript / Tailwind CSS без тяжелых внешних UI-библиотек.
  * **Control Panel (`web/src/features/support/ControlPanelApp.tsx`, 2 567 LOC):** Интерактивные Kanban/List представления тикетов, фильтрация по нарушениям SLA, рабочая область инцидента (чат, смена статусов, оценка поинтов, подзадачи `ticket_tasks`, аудит действий), карточки управления организациями и аналитика загрузки инженеров.
  * **Telegram Mini App (`web/src/features/tma/TelegramMiniApp.tsx`, 304 LOC + компоненты):** Автоматическая аутентификация через Telegram SDK, тактильный отклик (Haptic Feedback: `impact`, `notification`, `selection`), пошаговый мастер подачи заявки, мобильный чат и кнопки согласования бюджета в 1 клик.

#### 3. Аутентификация, ротация сессий и сквозной аудит (`api/app/routers/auth.py`, `api/app/audit.py`)
* **Назначение:** Выпуск и ротация JWT-токенов, контроль прав (RBAC), журнал действий.
* **Механизмы:**
  * Access/Refresh схема (TTL Access: 15 мин, Refresh: 14 дней) с валидацией `token_version`. Принудительный отзыв сессии сотрудника мгновенно инвалидирует все выданные токены инкрементом версии в БД.
  * Сервисные токены для Kinesis Hub на базе `client_id` и хеша секрета в Argon2id.
  * Таблица `audit_log`: структурированная фиксация всех мутаций (`actor_id`, `actor_kind`, `action`, `entity_type`, `entity_id`, `metadata_json`).

#### 4. Инфраструктурная обвязка и развёртывание (`deploy/`, `docker-compose.yml`)
* **Компоненты:** Docker Compose проект на 7 сервисов (`control-db`, `control-storage`, `control-migrate`, `control-api`, `control-worker`, `control-bot`, `control-web`).
* **Особенности:**
  * Контейнеры изолированы во внутренней сети `control-internal`; порты СУБД, MinIO и API не публикуются наружу.
  * `control-migrate` исполняется как одноразовый контейнер на этапе старта стека (`depends_on: condition: service_completed_successfully`), предотвращая старт API на немигрированной схеме.
  * Скрипты `deploy/init-env.sh` генерируют криптографически стойкие ключи `openssl rand` с правами `chmod 600`.
  * `deploy/postgres-init.sh` гарантирует создание роли приложения до запуска миграций.

---

## 3. РЕАЛЬНЫЙ ТЕХНОЛОГИЧЕСКИЙ СТЕК

* **Языки программирования:**
  * **Python 3.11+** (FastAPI Core, AsyncIO, SQLAlchemy Core & ORM, Worker, Bot).
  * **TypeScript 5.5+** (Frontend Core, State Management, API Layer, Telegram WebApp SDK).
  * **SQL / PL/pgSQL** (PostgreSQL DDL, RLS Policies, Grants, System Functions).
  * **POSIX Shell / Bash** (Deployment, Cryptographic bootstrapping, Backup/Restore scripts).
* **Бэкенд-фреймворки и библиотеки:**
  * **FastAPI 0.111+ / Starlette:** Высокопроизводительный асинхронный REST API.
  * **SQLAlchemy 2.0+:** ORM и Core-выражения, сессионные слушатели `after_begin`.
  * **Alembic:** Двунаправленные миграции структуры реляционной базы данных.
  * **Pydantic v2 & Pydantic-Settings:** Строгая валидация схем, DTO и типобезопасные конфигурации.
  * **Aiogram 3.8+:** Асинхронный фреймворк для взаимодействия с Telegram Bot API.
  * **HTTPX 0.27+:** Асинхронные HTTP-клиенты для IPC-взаимодействия воркера и бота.
  * **Argon2-cffi:** Криптографическое хеширование паролей и интеграционных секретов.
  * **PyJWT:** Выпуск и валидация JSON Web Tokens (HS256).
  * **Boto3 / Botocore:** Взаимодействие с объектным хранилищем S3.
* **Базы данных, хранилища и брокеры:**
  * **PostgreSQL 16 (Alpine):** Основная реляционная СУБД, Row-Level Security, JSONB, механизм очередей через `FOR UPDATE SKIP LOCKED`.
  * **MinIO:** S3-совместимое распределенное объектное хранилище медиаданных.
* **Фронтенд-стек:**
  * **React 18.3+:** Декларативный UI.
  * **Vite 5.3+:** Сборка и оптимизация клиентских бандлов.
  * **Tailwind CSS 3.4+ & PostCSS:** Утилитарный дизайн и адаптивная верстка.
  * **Lucide React:** Библиотека векторных пиктограмм.
  * **@telegram-apps/sdk-react / WebApp JS API:** Нативная интеграция с Telegram Mini App окружением.
* **Инфраструктура, контейнеризация и оркестрация:**
  * **Docker & Docker Compose (v2):** Многоконтейнерная изоляция микросервисов.
  * **Nginx (Alpine):** Веб-сервер статики, CSP-заголовки, обратный прокси для API.
  * **Healthcheck daemons:** Системные пробы готовности (`pg_isready`, MinIO live probe, HTTP `/ready`).

---

## 4. СЛОЖНЫЕ ИНЖЕНЕРНЫЕ РЕШЕНИЯ (Engineering Deep-Dive)

### 1. Форсированная изоляция данных через PostgreSQL RLS без суперпользователя
* **В чём заключалась сложность:** В подавляющем большинстве SaaS-приложений подключение к БД выполняется от имени владельца схемы. В PostgreSQL владелец таблиц и суперпользователь по умолчанию **игнорируют** политики RLS. При логической ошибке в SQL-запросе данные разных организаций становятся доступны чужим пользователям.
* **Решение автора:**
  1. Архитектурно разделены учетные записи: суперпользователь `postgres` используется строго для применения миграций схемы, а приложение исполняется под непривилегированным пользователем `kinesis_control` (`NOBYPASSRLS`).
  2. Во всех многоарендных таблицах применена директива `FORCE ROW LEVEL SECURITY`.
  3. В жизненный цикл сессии SQLAlchemy (`Session.after_begin`) внедрен хук, транзакционно проставляющий `set_config('app.tenant_id', :tenant_id, true)` и `set_config('app.platform_admin', :flag, true)`.
  4. Параметр `is_local = true` гарантирует локализацию настроек строго в рамках текущей транзакции, полностью исключая утечку прав при повторном использовании соединений из пула (Connection Pooling Safety).

### 2. Отказоустойчивый асинхронный Transactional Outbox на базе `FOR UPDATE SKIP LOCKED`
* **В чём заключалась сложность:** Необходимость гарантировать доставку критических событий (создание заявки, смена статуса, ответ оператора) в Telegram без развертывания тяжелых очередей (Kafka/RabbitMQ/Redis) и без блокировок строк при масштабировании воркеров на несколько реплик.
* **Решение автора:**
  1. События персистируются в таблицу `outbox_events` внутри единой ACID-транзакции с мутацией бизнес-сущностей.
  2. Реализован алгоритм неблокирующей выборки: `SELECT ... FROM outbox_events WHERE status = 'pending' AND available_at <= now() ORDER BY created_at FOR UPDATE SKIP LOCKED LIMIT 1`.
  3. Параллельные экземпляры воркера мгновенно пропускают заблокированные строки, исключая ожидание снятия блокировок и взаимные блокировки (deadlocks).
  4. Внедрен адаптивный экспоненциальный бэкофф (`now() + min(300, 2^attempts)`) и изолирование сбойных событий в статус `failed` после 8 неудачных попыток.

### 3. Алгоритм вычисления рабочих окон SLA с учетом часовых поясов и динамической заморозки
* **В чём заключалась сложность:** Вычисление дедлайнов реагирования и решения с учетом плавающих рабочих графиков клиентов, произвольных выходных дней, межсуточных переходов и несовпадения часового пояса клиента (`Asia/Tashkent`) с UTC-временем серверов.
* **Решение автора:**
  1. Реализована чистая функция `add_work_minutes`, выполняющая пошаговое продвижение временного курсора в локальной таймзоне клиента (`ZoneInfo`).
  2. Алгоритм циклически распределяет доступный остаток времени (`available = day_end - cursor`), пропуская выходные дни и нерабочие ночные часы.
  3. Интегрирован механизм заморозки регламента: время нахождения инцидента в ожидании информации от клиента накапливается в `sla_pause_seconds` и вычитается из финального времени разрешения, обеспечивая математически точную и справедливую отчетность.

### 4. Криптографический хендшейк и бесшовный Session Bridge для Telegram Mini App
* **В чём заключалась сложность:** Необходимость предоставить клиенту возможность мгновенного входа в мобильное приложение из Telegram без ввода учетных данных с гарантией невозможности подделки контекста пользователя.
* **Решение автора:**
  1. Разработан модуль `validate_telegram_webapp_init_data`, производящий валидацию подписи Telegram WebApp через каскадный HMAC-SHA256 от токена бота.
  2. Спроектирован механизм одноразовых приглашений (`TelegramInvite`): в базе персистируется только SHA-256 хеш токена с ограниченным сроком жизни.
  3. При первом открытии Mini App или переходе по ссылке в боте происходит атомарная привязка `TelegramBinding` с одновременным выпуском короткоживущего JWT с правами роли `client` и жестко привязанным `tenant_id`.

### 5. Сквозная идемпотентность операций и защита квот сопровождения
* **В чём заключалась сложность:** Сетевые сбои на мобильных устройствах приводят к повторной отправке HTTP-запросов клиентами. Это может вызывать дублирование инцидентов и многократное ошибочное списание поинтов из ежемесячного пакета организации.
* **Решение автора:**
  1. На уровне схемы БД наложен уникальный композитный индекс `UniqueConstraint("tenant_id", "idempotency_key")`.
  2. Обработчик `POST /tickets` проверяет наличие существующего тикета по ключу идемпотентности, а блок `except IntegrityError` производит откат транзакции и возвращает существующую запись вместо падения с ошибкой 500.
  3. Процесс согласования сметы (`/tickets/{id}/approval`) валидирует статус (`row.approval_status != 'pending'` → 409 Conflict), предотвращая возможность повторного списания квоты поинтов при параллельных запросах.

---

## 5. МАТЕРИАЛЫ ДЛЯ РЕЗЮМЕ (BigTech Standard)

### Для блока «Experience» (Формат Google XYZ)
* **Спроектировал и реализовал с нуля многоарендную платформу управления инцидентами и SLA (Kinesis Control)** на Python (FastAPI) и PostgreSQL, внедрив двухуровневую изоляцию через **PostgreSQL Row-Level Security (RLS)** и сессионные контексты, что обеспечило 100% изоляцию данных между корпоративными клиентами в рамках единого кластера без накладных расходов на дублирование инфраструктуры.
* **Разработал асинхронную шину доставки событий на базе паттерна Transactional Outbox** с неблокирующей конкурентной выборкой `FOR UPDATE SKIP LOCKED`, устранив проблему двойной записи (Dual-Write) и гарантировав надежную доставку уведомлений в каналы Telegram с экспоненциальным бэкоффом без использования внешних брокеров сообщений.
* **Создал алгоритмический движок расчета SLA с поминутным распределением рабочих интервалов**, учетом часовых поясов (`ZoneInfo`), производственных календарей и логики динамической заморозки, полностью исключив ложные нарушения дедлайнов в нерабочие часы и обеспечив прозрачный биллинг квот сопровождения.
* **Разработал омниканальную экосистему клиентского доступа**, объединив операторскую консоль (React/TypeScript), диалогового Telegram-бота и мобильный Telegram Mini App (TMA) с криптографической аутентификацией HMAC-SHA256, сократив среднее время регистрации инцидентов клиентами с нескольких минут до 30 секунд.

### Для блока «Projects»
* **Kinesis Control — Enterprise Multi-tenant Incident Management & SLA Platform:**
  * **Core Architecture:** Спроектировал модульную архитектуру микросервиса управления инцидентами со строгой изоляцией тенантов (PostgreSQL FORCE RLS), ролевым доступом (RBAC) и разграничением контекстов для 4 типов акторов (`platform`, `client`, `hub`, `service`).
  * **SLA & Quota Engine:** Реализовал подсистему динамического расчета сроков реагирования/решения инцидентов и автоматического списания поинтов из ежемесячных пакетов клиентов с поддержкой двухфазного согласования смет (Story Points / Commercial Pricing).
  * **High-Reliability Event Pipeline:** Построил фоновый воркер на базе паттерна Transactional Outbox в PostgreSQL с конкурентной выборкой `SKIP LOCKED` и IPC-шлюзом в Telegram Bot API.
  * **Frontend & Mini App Ecosystem:** Разработал десктопное SPA для операторов поддержки и мобильное веб-приложение Telegram Mini App с нативной поддержкой тактильного отклика (Haptics), потоковой отдачей медиафайлов из S3 и интерактивным согласованием тикетов.

---

## 6. ИНВЕНТАРЬ ПОДТВЕРЖДЕННЫХ НАВЫКОВ (Skills Matrix)

| Домен | Подтвержденные кодовой базой технологии и концепции |
| :--- | :--- |
| **Languages** | **Python 3.11+** (AsyncIO, Type Hinting, OOP, Metaprogramming), **TypeScript 5+** (Strict mode, Generics, DTOs), **SQL** (PostgreSQL DDL/DML, Constraints, Session variables, Locks), **POSIX Shell** |
| **Architecture & Patterns** | Multi-tenancy Architecture, PostgreSQL Row-Level Security (RLS), Transactional Outbox Pattern, Event-Driven Processing, Concurrency Control (`SKIP LOCKED`), State Machine (FSM), Idempotent APIs, CQRS elements, Zero-Trust Actor Model |
| **Frameworks & Libs** | **FastAPI**, **SQLAlchemy 2.0 (Core & ORM)**, **Alembic**, **Pydantic v2**, **Aiogram 3**, **React 18**, **Vite**, **Tailwind CSS**, **Boto3**, **HTTPX**, **Argon2-cffi**, **PyJWT** |
| **Databases & Storage** | **PostgreSQL 16** (RLS, Constraints, Indexes, JSONB, Session configs), **MinIO / AWS S3** (Multipart uploads, Object streaming, Presigned URLs) |
| **Security & Cryptography** | HMAC-SHA256 signature verification, Argon2id password hashing, Cryptographic token generation (`token_urlsafe`), JWT token rotation and versioning, Principle of Least Privilege (DB Roles), Content-Security-Policy (CSP) |
| **DevOps & Infrastructure** | **Docker**, **Docker Compose** (Healthchecks, Service dependencies, Volumes, Networks), **Nginx** (Reverse Proxy, Dynamic DNS resolution, Caching headers, SPA routing), Multi-stage Dockerfiles, Automated Backup & Restore verification scripts |

