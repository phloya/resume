# ТЕХНИЧЕСКИЙ АУДИТ И ИНЖЕНЕРНОЕ ДОСЬЕ СИСТЕМЫ «KINESIS HUB & AVA TELEPHONY PLATFORM»

---

## 1. МАКРО-УРОВЕНЬ (Система в целом)

### Назначение системы
**Kinesis Hub & AVA** — распределенная омниканальная SaaS-платформа корпоративного уровня для оркестрации автономных AI-агентов (текстовых и голосовых в реальном времени), омниканальных бизнес-коммуникаций, CRM-процессов, динамического управления базами знаний (RAG, ERP/1С:Предприятие, складской учет, слоты бронирования) и сквозной аналитики качества диалогов. 

Платформа решает задачу полной автоматизации первого и последующих кругов клиентского взаимодействия (продажи, поддержка, консультирование, выставление счетов, назначение встреч, реактивация лидов и холодные/теплые обзвоны) с мгновенной эскалацией на операторов-людей при возникновении критических триггеров.

### Масштаб кодовой базы и структуры
- **Суммарный объем кодовой базы:** **465 441 строк кода (LOC)** в **1 201 файле** (без учета `node_modules`, `.git`, виртуальных окружений и бинарных ассетов).
- **Распределение по языкам программирования:**
  - **Python:** 568 файлов, **270 712 строк** (доменная логика, CRM backend, LangGraph-оркестратор, voice/telephony engine, медиасервисы, адаптеры интеграций).
  - **TypeScript / React:** 336 файлов, **111 667 строк** (интерфейс Kinesis Hub: дизайн-система «Тихая диспетчерская», операционные пульты, E2E тесты Playwright, хранилища состояний).
  - **Shell (Bash):** 19 файлов, **10 735 строк** (продакшн-инсталляторы, скрипты развертывания телефонии, автоматическая генерация PJSIP-конфигураций Asterisk, оркестрация Docker).
  - **Markdown (Техническая и архитектурная документация):** 161 файл, **44 834 строки** (контракты API, карты сервисов и рисков, дизайн-система, процедуры аварийного восстановления).
  - **Конфигурации и схемы (JSON, YAML, Dockerfile, Conf):** 117 файлов, **27 493 строки** (декларация 17 контейнеров Docker Compose, multi-stage сборка, Nginx-проксирование, PJSIP/Asterisk dialplans).
- **Сложность структуры каталогов:** Монорепозиторий с микросервисной декомпозицией на **14 независимых вычислительных подсистем**, каждая из которых изолирована в собственном контейнере, обладает своим жизненным циклом, стратегиями персистентности и спецификациями API.

### Архитектурный паттерн
- **Distributed Microservices / Service-Oriented Architecture (SOA)** с гибридной моделью взаимодействия:
  - **Синхронный контур (Real-Time Ingestion):** HTTP/REST с Circuit Breaker и строгими таймаутами для клиентских запросов и валидации пайплайнов.
  - **Асинхронный событийный контур (Event-Driven):** Фоновые задачи (FastAPI `BackgroundTasks`, Redis Priority Queues с lease-механизмами, PostgreSQL `SELECT FOR UPDATE SKIP LOCKED` для распределенных воркеров).
  - **Real-Time Reactive Pub/Sub:** Механизм `LISTEN / NOTIFY` (`pg_notify`) PostgreSQL на стороне ядра базы данных, транслируемый демоном в `RealtimeHub` на WebSocket-пул фронтенда без накладных расходов на внешний брокер.
  - **State Machine & Graph Execution:** Стейт-машины на базе **LangGraph** (ReAct-паттерн с динамической компиляцией графа и контрольными точками в PostgreSQL) и стейт-машины жизненного цикла лидов/сделок в CRM.

### Системные гарантии
- **Concurrency & High Availability:** 
  - Горизонтальное масштабирование API-шлюза: пул реплик `backend-ui` и `backend-ui-replica` под балансировкой Nginx по алгоритму `least_conn`.
  - Четкое разделение первичных и вторичных ролей: фоновые периодические воркеры (биллинг, 1C-синхронизация, spam timeout) активны строго на ведущей ноде (`PAYMENTS_WORKER_PRIMARY=true`), исключая split-brain и дублирование cron-задач.
  - Распределенный координатор блокировок: предотвращение конфликтов long-polling в Telegram Bot API через распределенный Redis-лиз с атомарными Lua-скриптами (`SET NX EX` + renewal).
- **Консистентность данных:** Идемпотентность всех побочных эффектов (Idempotency Keys в таблице `crm_tool_action_runs` и `crm_payment_provider_events`), транзакционная изоляция сессий SQLAlchemy, дедупликация входящих вебхуков по составным ключам `(project_id, origin, external_message_id)`.
- **Изоляция и Multi-Tenancy:** Логическое разделение на уровне `project_id` и `agent_id` по всем таблицам БД, метаданным RAG-векторов pgvector и контекстам выполнения инструментов.

### Рекомендуемый тайтл автора
**Lead AI Systems Architect / Staff Backend & AI Infrastructure Engineer**.  
*Обоснование:* Автор спроектировал и реализовал с нуля распределенный стек с глубоким стыком Enterprise-бэкенда (CRM, биллинг, ERP/1С:Предприятие, RBAC), сложной AI-оркестрации (LangGraph, токенизация, превентивная компиляция графов, pgvector RAG) и телекоммуникационной инфраструктуры реального времени (Asterisk ARI, AudioSocket TCP, ExternalMedia RTP UDP, barge-in, SIP trunking).

---

## 2. АРХИТЕКТУРНАЯ ДЕКОМПОЗИЦИЯ С ПРИОРИТИЗАЦИЕЙ

### ФУНДАМЕНТАЛЬНЫЕ ПОДСИСТЕМЫ (75–80% архитектурного ядра)

#### 1. CRM Core Engine & State Layer (`Json_servise/app/crm/`)
- **Назначение:** Центральное ядро бизнес-логики, персистентности, нормализации сущностей, управления воронками, скоринга и реактивного оповещения.
- **Схема данных и транзакционная модель:**
  - Реализовано более 35 сущностей SQLAlchemy ORM (`Json_servise/app/crm/orm.py`):
    - *Субъекты и доступ:* `Contact` (уникальность по `project_id + phone_normalized`), `Lead`, `Conversation`, `ChannelLink`, `CrmUser`, `Manager`, `AccessRole`, `OperatorRole`.
    - *Воронки и процессы:* `Deal`, `DealStageEvent`, `CrmTask`, `Escalation`, `OperatorNote`.
    - *Омниканальность и события:* `MessageEvent`, `MediaAsset`, `MediaCleanupRun`, `DeletionTombstone`, `DeletionCleanupJob`.
    - *Финансы и платежи:* `PaymentInvoice`, `PaymentProviderEvent`, `AgentPaymentConfig`.
    - *Знания и интеграции:* `KnowledgeDocument`, `WarehouseItem`, `SlotItem`, `DynamicKnowledgeState`, `WarehouseDataServiceSyncConfig`.
    - *Рассылки и телеком:* `BroadcastCampaign`, `BroadcastRecipient`, `BroadcastDeliveryAttempt`, `BroadcastThrottle`, `TelephonyOutboundCampaign`, `TelephonyOutboundRecipient`.
- **Алгоритмы и механизмы синхронизации:**
  - **Identity Resolution & Contact Merging:** Алгоритм дедупликации контактов (`get_or_create_contact`, `merge_contact_into_contact`). При поступлении нового номера телефона или внешнего ID из канала происходит автоматическое слияние каналов, перепривязка сделок, диалогов и системных событий без потери истории.
  - **Реактивный WebSocket-броадкаст через PostgreSQL:** В `realtime.py` реализована архитектура `RealtimeHub`. Вместо отдельного Redis Pub/Sub задействован нативный `pg_notify(channel, payload)`. Фоновый поток Python слушает `LISTEN crm_realtime` через `psycopg2` и `select.select`, транслируя события в асинхронный event loop через `asyncio.run_coroutine_threadsafe()`. Для обхода лимита PostgreSQL NOTIFY в 8000 байт реализовано автоматическое усечение полезной нагрузки с флагом `truncated: True`, инициирующее точечный refetch сущности клиентом.
  - **Worker Claiming через Row-Level Locks:** В обработчике платежей (`payments/worker.py`) и очередях рассылок захват задач воркерами выполняется через `SELECT ... FOR UPDATE SKIP LOCKED` с обновлением `lease_owner` и `lease_until`. Это гарантирует отсутствие race conditions и взаимоблокировок (deadlocks) между репликами бэкенда.
  - **GDPR / Soft Deletion & Tombstones:** Механизм `DeletionTombstone` и фоновых джоб `DeletionCleanupJob` гарантирует отложенную асинхронную зачистку файлов и связанных сущностей при сохранении целостности внешних идентификаторов.
- **Кастомная разработка vs библиотеки:**
  - *Кастомное (100% авторский код):* Вся доменная логика слияния контактов, пайплайны стадий сделок, машина эскалаций, WebSocket-мост через `pg_notify`, логика захвата задач с лизами, алгоритмы валидации и нормализации номеров телефонов.
  - *Библиотеки:* SQLAlchemy 2.0 (ORM/Core), Psycopg2-binary, Pydantic v2, FastAPI.

#### 2. AI Orchestrator & Graph Runtime (`LLM service/`)
- **Назначение:** Мультиагентный вычислительный движок на базе ReAct, управляющий контекстом, динамическим выбором инструментов, персистентными контрольными точками и памятью диалогов.
- **Архитектурные механизмы:**
  - **LangGraph Checkpointing & Isolation:** Использование `create_react_agent` с персистентным чеклистом в отдельной базе данных PostgreSQL (`db-langgraph`). При каждом запуске сессии генерируется изолированный `thread_id`. По завершении хода чекпоинт очищается через `_clear_thread_checkpoint()`, гарантируя, что единственным источником правды для истории диалога остается база CRM, предотвращая рассинхронизацию стейта.
  - **Dynamic State & Pydantic Model Generation:** Функция `_create_dynamic_output_schema()` на лету динамически конструирует Pydantic-модели с типизированными литералами `Literal[*state_keys]` на основе актуальных настроек агента из CRM.
  - **Pre-Model Trim Hook & Токено-зависимая компактизация контекста:** Модуль `strategies.py` реализует пре-хук `pre_model_trim_hook` перед вызовом нейросети:
    - Вычисляет точный объем контекста через токенизатор BPE `tiktoken` (`cl100k_base`).
    - Автоматически отсекает осиротевшие сообщения инструментов (`_drop_leading_orphan_tool_messages`), предотвращая поломку API провайдеров (`tool_call_id` without matching `tool`).
    - При приближении к порогу контекста (настраивается метаданными агента, по умолчанию 900k токенов) изолирует хвост из последних $N$ сообщений пользователя (`_recent_client_tail_start`), а более раннюю историю передает во вторичную LLM (`LLM_CONTEXT_SUMMARY_MODEL`, напр. `gpt-oss-20b`) для генерации концентрированного фактологического экстракта.
    - Реализован двухуровневый LRU-кэш саммари (`_SUMMARY_CACHE`) с SHA-256 хэшированием списка сообщений.
  - **Кэширование графа с автоматической инвалидацией:** В `main.py` граф компилируется и кэшируется по `agent_id`. Перед выполнением вычисляется SHA-256 fingerprint отпечатка конфигурации (системный промпт, модель, подключенные тулы, состояния). При изменении настроек в CRM граф прозрачно перекомпилируется без перезапуска процесса.
- **Кастомная разработка vs библиотеки:**
  - *Кастомное:* Алгоритм токено-зависимой компактизации контекста, хуки очистки осиротевших tool-сообщений, хэширование конфигураций для инвалидации графа, генераторы динамических схем вывода, фильтрация внутренних служебных префиксов (`INTERNAL:`).
  - *Библиотеки:* LangGraph, LangChain Core, LangChain OpenAI, Tiktoken.

#### 3. Execution & Action Subsystem (`Tool Execution Service/`)
- **Назначение:** Высоконадежное исполнение бизнес-действий, инициированных AI-агентом (отправка медиафайлов, создание задач менеджерам, изменение стадий воронок, выставление счетов, бронирование слотов, вызов омниканальных API).
- **Паттерны надежности и конкурентность:**
  - **Idempotency Gate (`crm_tool_action_runs`):** Перед выполнением side-effect действия сервис выполняет claim в таблице `crm_tool_action_runs` по составному `idempotency_key` (`conversation_id + action_type + rule_id/hash`). Если действие уже находится в статусе `processing` или `completed`, повторный запуск блокируется, предотвращая дублирование отправок файлов или повторное создание задач.
  - **Asynchronous Decoupling (Non-blocking vs Blocking):** Бизнес-действия (`TaskManager`, `LeadContacts`, `StageManager`, `Escalation`, `SmartFollowUp`) мгновенно возвращают статус `accepted` в LangGraph, освобождая LLM для генерации ответа клиенту, а само действие выполняется в фоновом таске `asyncio.create_task`. Действие `FileSender` выполняется синхронно-блокирующе, чтобы агент знал фактический факт успешности отдачи медиафайла.
  - **Circuit Breaker для зависимых сервисов:** В `main.py` реализован конечный автомат Circuit Breaker (`TOOL_OPTIONAL_CIRCUIT_FAILURES`, `TOOL_OPTIONAL_CIRCUIT_RESET_SECONDS`) для внешних шлюзов (`telegramclient`, `telegrambot`, `followup-service`, `umnico`, `bitrix`). При серии сетевых сбоев контур размыкается, предотвращая лавинообразные каскадные зависания потоков.
  - **Graceful Shutdown:** Регистрация всех активных задач в `background_action_tasks` с ожиданием завершения в течение 10 секунд при получении сигнала завершения контейнера.
- **Кастомная разработка vs библиотеки:**
  - *Кастомное:* Схема идемпотентного клейма, Circuit Breaker на базе `time.monotonic()`, адаптеры формирования подписанных ссылок, мапперы системных событий.
  - *Библиотеки:* FastAPI, HTTPX, Pydantic.

#### 4. Distributed Priority Follow-Up Scheduler (`schedular service /`)
- **Назначение:** Распределенный планировщик отложенных касаний, цепочек сообщений и отложенных хуков генерации саммари с учетом часовых поясов и рабочего времени.
- **Управление состоянием и алгоритмы очередей:**
  - **Redis-Backed Queues с Lease-механизмом:** Очередь построена на базе Redis Sorted Sets (ZSET), где score — Unix timestamp момента выполнения (`due_time`).
  - **Three-Stage Job Processing:**
    1. Фоновый опрос ZSET каждые 2 секунды пачками (`TASK_FETCH_BATCH_SIZE=50`).
    2. Атомарный перенос задачи из `scheduled` в `processing` с назначением лиза (`TASK_LEASE_SECONDS=120`).
    3. При успешном выполнении — удаление деталей задачи (`del`).
    4. При сбое — экспоненциальный retry/backoff с возвратом в очередь до `TASK_MAX_RETRIES=5`, после чего задача отправляется в Dead-Letter Queue (DLQ).
  - **Timezone- & Working-Hours-Aware Scheduling:** Вычисление времени срабатывания с жесткой проверкой рабочих часов бизнеса (по умолчанию 09:00–21:00 с поддержкой любых IANA-таймзон, например `Asia/Tashkent`). Если целевое время выпадает на нерабочий интервал или ночь, задача автоматически смещается на утро следующего разрешенного дня.
  - **Dynamic Sliding Summary Posthook:** Автоматическое планирование вызова LLM для суммаризации диалога через 60 минут после последнего входящего сообщения. Каждое новое сообщение от клиента сдвигает таймер вперед, предотвращая преждевременную суммаризацию активной беседы.
- **Кастомная разработка vs библиотеки:**
  - *Кастомное:* Весь воркер планировщика, алгоритм смещения касаний по рабочим часам, скользящий таймер саммари, логика отмены серии при получении opt-out фразы от клиента.
  - *Библиотеки:* Redis-py (asyncio), FastAPI, HTTPX, Python-dateutil.

#### 5. Real-Time Telephony & Voice Agent Engine (`TelephonyService/Telephony/` — AVA Platform)
- **Назначение:** Высокопроизводительная телеком-платформа для проведения голосовых телефонных звонков с AI в режиме реального времени поверх IP-АТС Asterisk/FreePBX.
- **Низкоуровневая архитектура и обработка аудио:**
  - **Asterisk ARI & Transports:** Управление вызовами через Asterisk REST Interface (ARI). Поддержка двух параллельных архитектур аудиотранспорта:
    - **AudioSocket (TCP):** Потоковая двунаправленная передача 16-bit PCM аудио без джиттера.
    - **ExternalMedia RTP (UDP):** Прямой RTP-стриминг с минимальной задержкой сетевого стека через `rtp_server.py`.
  - **Barge-In & Conversation Coordination:** Модули `ConversationCoordinator`, `AudioGatingManager` и `VADManager`. Непрерывный мониторинг входящего аудиопотока абонента. При детекции голоса говорящего абонента во время ответа AI текущее воспроизведение мгновенно прерывается (barge-in), буфер воспроизведения очищается, а контекст прерывания фиксируется в сессии.
  - **Провайдеры реального времени:** Поддержка монолитных Realtime Full-Duplex протоколов через WebSockets (OpenAI Realtime API, Google Gemini Live / Vertex AI Live, Deepgram Voice Agent, ElevenLabs Conversational AI) и модульных конвейеров (отдельные STT -> LLM -> TTS звенья).
  - **Интеграция с SaaS CRM:** Модуль `telephony_bridge.py` транслирует каждый звонок в сущности CRM: стенограмма распознавания, длительность, аудиозапись и результаты звонка связываются с контактом и лидом, отображаясь в живом таймлайне интерфейса.
- **Кастомная разработка vs библиотеки:**
  - *Кастомное:* ARI-клиент, AudioSocket-сервер, RTP-сервер, координатор прерываний (barge-in), VAD-менеджер, ресемплер аудиопотоков, интеграционный мост с CRM.
  - *Библиотеки:* WebSockets, PyPJSIP/Asterisk, NumPy (аудиоманипуляции).

---

### ВСПОМОГАТЕЛЬНЫЕ ПОДСИСТЕМЫ (20–25% архитектуры)

#### 6. Enterprise ERP & Warehouse Synchronizer (`Json_servise/app/logic/onec_client.py`)
- **Задача:** Двусторонняя периодическая и триггерная синхронизация номенклатуры, остатков по складам и цен с 1С:Предприятие (Alphakassa API) и сопоставление с каталогом изображений WooCommerce (`tanvir.uz`).
- **Контракты и безопасность:**
  - **Криптографическая защита:** Пароли учетных записей 1С шифруются с использованием **AES-256-GCM (AEAD)** (`onec_credentials.py`). Контекст шифрования связывается с метаданными (`_associated_data(project_id, agent_id)`), исключая возможность подмены шифротекста между проектами.
  - **Динамический маппинг полей:** Поддержка вложенных путей dot-notation (`product.price`, `brand.name`) и вычисление `row_hash` для обновления только изменившихся позиций.

#### 7. RAG Knowledge Vector Engine (`RAG Service /`)
- **Задача:** Векторизация загружаемых корпоративных регламентов и семантический поиск фрагментов для ответов агентов.
- **Контракты и хранилище:**
  - База данных: PostgreSQL 16 с расширением `pgvector`.
  - Модель векторного хранилища: `PGVectorStrategy` с поддержкой `JSONB` метаданных.
  - Асинхронный пайплайн индексации: метод возвращает HTTP 202, индексация идет в фоне через `RecursiveCharacterTextSplitter` (chunk size: 1000, overlap: 200) с отправкой callback-статуса (`processing -> ready / failed`) в CRM.
  - Изоляция тенантов: фильтрация поиска по `project_id`, `agent_id` и белому списку документов.

#### 8. Omnichannel Gateways (Telegram Userbot, Telegram Bot, Umnico, Bitrix24)
- **Telegram Userbot (`TelegramService`):** Telethon (MTProto), поддержка сессий, буферизация коротких сообщений с дебаунсом (`response_delay_seconds`), загрузка медиафайлов.
- **Telegram Bot Service (`TelegramBotService`):** Aiogram 3, long-polling с распределенным Redis-лизом (`PollingLeaseCoordinator`), автоматическое извлечение и валидация координат локаций (Point/Venue/Live) с формированием ссылок на геосервисы (Google, Yandex, OSM).
- **Bitrix24 Open Lines Gateway (`BitrixSystem`):** Глубокая интеграция с Открытыми линиями и CRM Битрикс24. Полноценное сканирование сделок, проверка прав оператора, загрузка файлов через Disk API (`im.disk.folder.get` -> `disk.folder.uploadfile` -> `im.disk.file.commit` с флагом `SILENT_MODE=Y`).
- **Umnico Gateway (`Umnico service`):** Обработка входящих вебхуков WhatsApp, Instagram, Telegram с роутингом по `lead_id` и буферизацией.

#### 9. Multimodal Audio & Vision Engine (`PhotoVoice/`)
- **Задача:** Распознавание голосовых сообщений (Whisper STT) и визуальный анализ входящих изображений (OpenAI Vision / GPT-4o-mini).
- **Контракт:** Принимает multipart media, возвращает нормализованный текст и дескрипторы для инъекции в диалоговый контекст агента.

#### 10. Spam Shield Gateway (`Spam Service/`)
- **Задача:** Высокоскоростной синхронный пре-скоринг входящих сообщений (regex, стоп-слова, эвристики ссылок) с таймаутом до 4 секунд, предотвращающий лишние вызовы дорогих LLM.

#### 11. Business Intelligence & Quality Analytics (`Analytics Service/`)
- **Задача:** Фоновый сбор аналитических срезов, расчет конверсий воронок, аудит соблюдения скриптов и запуск вторичных LLM для оценки качества диалогов (Quality Assurance scoring).

---

## 3. РЕАЛЬНЫЙ ТЕХНОЛОГИЧЕСКИЙ СТЕК

- **Языки программирования:** Python 3.11+, TypeScript 5+, Go 1.22+ (CLI телефонии), POSIX Bash, SQL.
- **Backend & AI Frameworks:**
  - FastAPI, Starlette, Uvicorn, Gunicorn.
  - LangChain Core, LangGraph, Tiktoken.
  - SQLAlchemy 2.0 (Core & ORM), Alembic, Pydantic v2.
  - Aiogram 3 (Bot framework), Telethon (MTProto Telegram Client).
- **Frontend & UI:**
  - React 18, Vite, TypeScript.
  - Tailwind CSS, Lucide React, Radix UI.
  - Playwright (E2E testing), Vitest (Unit testing).
- **Базы данных, кэш и хранилища:**
  - **PostgreSQL 16** (CRM DB, LangGraph Checkpoints DB, pgvector Vector DB).
  - **pgvector** (HNSW/IVFFlat векторные индексы).
  - **Redis 7 (Alpine)** (Очереди планировщика, распределенные лизы, кэш).
  - **SQLite 3** (Локальное хранилище телефонии `agents.db`, `call_history.db`, локальная аналитика).
  - Docker Volumes (локальное файловое хранилище подписанных ассетов).
- **Телефония и Real-Time Media:**
  - Asterisk 20+ (PJSIP, ARI — Asterisk REST Interface).
  - AudioSocket Protocol (TCP PCM 16-bit).
  - ExternalMedia RTP (UDP audio streaming).
  - Local AI Engine: Vosk, Sherpa-ONNX, Piper TTS, Kokoro TTS, llama.cpp.
- **Криптография и протоколы:**
  - AES-256-GCM (AEAD через Python `cryptography`).
  - HMAC-SHA256 (Signed Asset URLs, Auth Tokens `kh1.`).
  - WebSocket (Full-Duplex CRM synchronization, Realtime AI Voice).
- **Инфраструктура, DevOps и мониторинг:**
  - Docker, Docker Compose (17 сервисных контейнеров).
  - Nginx (Edge reverse proxy, SSL termination через Let's Encrypt / ACME Companion, балансировщик `least_conn`).
  - Systemd, Linux cgroups, Host Networking (для RTP/SIP задержек).

---

## 4. СЛОЖНЫЕ ИНЖЕНЕРНЫЕ РЕШЕНИЯ (Engineering Deep-Dive)

### 1. Распределенный координатор конкурентного Long-Polling без конфликтов Telegram API
- **Проблема:** При горизонтальном масштабировании инстансов или перезапуске контейнеров `telegrambot` несколько процессов одновременно пытаются запустить `getUpdates` для одного и того же bot token. Telegram API немедленно возвращает ошибку `409 Conflict: terminated by other getUpdates request`, приводя к циклическому падению процесса (crash loopback) и потере сообщений.
- **Решение:** В `TelegramBotService/polling_lease.py` спроектирован координатор распределенного владения лизом на базе Redis (`PollingLeaseCoordinator`). Захват, продление и освобождение лиза реализованы через атомарные Lua-скрипты:
  ```lua
  if redis.call('get', KEYS[1]) == ARGV[1] then
    return redis.call('expire', KEYS[1], ARGV[2])
  end
  return 0
  ```
  Внедрен `POLLING_WATCHDOG_TASK`, непрерывно мониторящий здоровье поллинга. Если инстанс теряет лиз или сталкивается с сетевым зависанием более 90 секунд, он корректно передает право владения резервной ноде без перезапуска всего микросервиса.

### 2. Токено-зависимая компактизация контекста диалога с защитой структуры графа
- **Проблема:** В длинных диалогах (более 30–50 сообщений) размер контекста превышает эффективное окно модели, вызывая экспоненциальный рост стоимости и деградацию качества ответов. Простая обрезка по количеству сообщений (FIFO) ломает протокол OpenAI/LangGraph, если случайно удаляется `AIMessage` с `tool_calls`, оставляя следующий за ним `tool` без родителя, что вызывает фатальный `BadRequestError` от API.
- **Решение:** В `LLM service/strategies.py` внедрен специализированный хук `pre_model_trim_hook`:
  1. Выполняется точный подсчет токенов через BPE-кодировщик `tiktoken`.
  2. Реализована функция `_drop_leading_orphan_tool_messages()`, которая гарантирует целостность пар вызовов инструментов при любой обрезке.
  3. Контекст делится на «хвост недавних реплик» (`_recent_client_tail_start`, сохраняющий последние $N$ сообщений пользователя в исходном виде) и «историческую базу».
  4. Историческая база асинхронно сжимается через вторичную модель-суммаризатор в фактологическую сводку («Краткая память предыдущей части диалога»), которая кэшируется по SHA-256 хэшу от исходного массива сообщений. Это обеспечивает непрерывность контекста при строгом контроле токенов.

### 3. Реактивная синхронизация интерфейса без внешнего брокера очередей (PostgreSQL Listen/Notify Bridge)
- **Проблема:** Требовалось реализовать real-time отображение сообщений, смены статусов лидов и звонков в веб-интерфейсе для сотен операторов. Использование внешнего брокера (RabbitMQ/Redis PubSub) усложнило бы архитектуру и требовало бы двухфазных коммитов для синхронизации записи в БД и публикации в сокет.
- **Решение:** В `Json_servise/app/crm/realtime.py` реализован мост на базе встроенных механизмов СУБД:
  1. В рамках единой транзакции SQLAlchemy выполняется функция `publish_realtime_event()`, вызывающая нативный `SELECT pg_notify('crm_realtime', payload)`. Событие генерируется строго в момент фиксации транзакции в Postgres.
  2. Выделенный демон-поток в Python поддерживает постоянное соединение через `psycopg2` и слушает канал через системный вызов `select.select`.
  3. Полученные уведомления диспетчеризируются в асинхронный event loop через `asyncio.run_coroutine_threadsafe()` в пул соединений `RealtimeHub`.
  4. Для обхода жесткого лимита PostgreSQL NOTIFY (8000 байт) внедрен алгоритм безопасного сжатия: если JSON превышает 7800 байт, тело сообщения усекается до идентификатора сущности с флагом `truncated: True`, что сигнализирует фронтенду о необходимости фонового дозапроса через REST API.

### 4. Криптографически изолированное хранение учетных данных внешних ERP (1С)
- **Проблема:** Multi-tenant SaaS требует хранения паролей от API 1С:Предприятие сотен различных контрагентов. Хранение в открытом виде недопустимо, а симметричное шифрование единым ключом уязвимо к подмене зашифрованных токенов между проектами (ciphertext swapping attack).
- **Решение:** В `Json_servise/app/logic/onec_credentials.py` реализована аутентифицированная схема шифрования **AES-GCM (AEAD)**:
  - Из мастер-ключа через SHA-256 деривируется ключ шифрования.
  - На каждый пароль генерируется случайный криптографический вектор инициализации (nonce) длиной 12 байт (`os.urandom(12)`).
  - В качестве дополнительных аутентифицированных данных (Associated Data) передается жесткая привязка к тенанту: `kinesis-hub:onec:{project_id}:{agent_id}:password`.
  - При попытке скопировать зашифрованный токен в профиль другого агента или проекта расшифровка немедленно падает с ошибкой `InvalidTag`, полностью исключая утечку или подмену доступа.

### 5. Полнодуплексный голосовой конвейер с подавлением воспроизведения (Barge-In) для телефонии
- **Проблема:** В телефонном разговоре задержка между репликой абонента и ответом AI должна составлять менее 600–800 мс. Если абонент перебивает говорящего бота, бот обязан немедленно замолчать, иначе диалог превращается в какофонию.
- **Решение:** В AVA Telephony Engine (`TelephonyService/Telephony/src/core/`):
  - Построен двунаправленный стриминг PCM-аудио поверх TCP AudioSocket и UDP RTP.
  - Реализован `AudioGatingManager` совместно с `VADManager`: входящий голосовой поток абонента анализируется фреймами по 20 мс.
  - При превышении порога вероятности речи (VAD speech threshold) во время активного состояния воспроизведения ответа AI генерируется мгновенный сигнал прерывания: буфер воспроизведения `PlaybackManager` сбрасывается, на АТС Asterisk отправляется команда остановки воспроизведения, а текущий WebSocket-стрим к провайдеру (напр. OpenAI Realtime) прерывается отменой фрейма (`response.cancel`), переводя систему в режим слушания.

---

## 5. МАТЕРИАЛЫ ДЛЯ РЕЗЮМЕ (BigTech-Style)

### Блок «Experience» (Системный масштаб и метрики)
- **Спроектировал и разработал распределенную SaaS-платформу омниканальных AI-агентов и голосовой телефонии объемом 465k+ LOC на Python, TypeScript и Asterisk**, объединившую текстовые каналы (Telegram, WhatsApp, Instagram, Bitrix24), голосовой стриминг реального времени и ERP (1С:Предприятие), что сократило время обработки входящих лидов с десятков минут до субсекундного отклика.
- **Разработал отказоустойчивое ядро оркестрации AI-агентов на базе LangGraph с динамической Pydantic-схемогенерацией и пре-модельным BPE-токенизатором**, внедрив превентивную компиляцию графов и семантическое сжатие контекста, что снизило затраты на токены на ~40% и полностью предотвратило падения сессий из-за превышения контекстного окна в длительных диалогах.
- **Спроектировал высоконадежную систему очередей и распределенного планирования на базе PostgreSQL (Row-Level Locking `SKIP LOCKED`), Redis ZSET и Lua-скриптов**, обеспечив строгие гарантии доставки отложенных триггеров с учетом рабочих часов и IANA-таймзон, а также устранив race conditions и дублирование polling-сессий Telegram при горизонтальном масштабировании.
- **Реализовал сверхнизколатентный голосовой шлюз для интеграции Asterisk ARI с WebSockets AI-провайдерами (OpenAI Realtime, Google Live, Deepgram)** через AudioSocket (TCP) и ExternalMedia RTP (UDP) с аппаратным VAD и прерыванием речи (barge-in), обеспечив задержку диалога менее 700 мс в реальных телефонных сетях.

### Блок «Projects» (Архитектурные компоненты и фичи)
- **Kinesis Hub Full-Stack Control Plane:** Разработал модульный рабочий интерфейс оператора и администратора (React 18, Vite, Tailwind CSS, Playwright) с собственной дизайн-системой, реактивным WebSocket-стримингом событий через связку PostgreSQL `pg_notify` / FastAPI, управлением воронками сделок и динамическими карточками лидов.
- **Autonomous Tool Execution Subsystem:** Создал микросервис изолированного выполнения инструментов AI с распределенной защитой от дублирования вызовов (Idempotency Keys), автоматическим Circuit Breaker для сторонних API и асинхронным decoupling-конвейером для фоновых задач CRM.
- **Secure Enterprise 1C / Warehouse Bridge:** Спроектировал конвейер двусторонней синхронизации номенклатуры и остатков складов с криптографической защитой учетных данных на базе AES-256-GCM (AEAD с жесткой привязкой к тенанту) и сопоставлением медиа-каталога WooCommerce.
- **Multi-Tenant RAG Engine:** Развернул векторную поисковую подсистему на базе PostgreSQL 16 + pgvector, реализовав асинхронную чанко-векторизацию документов в фоне с callback-уведомлением статуса и строгой логической изоляцией тенантов по метаданным.

---

## 6. ИНВЕНТАРЬ ПОДТВЕРЖДЕННЫХ НАВЫКОВ (Skills Matrix)

| Категория | Технологии и паттерны, подтвержденные кодовой базой |
| :--- | :--- |
| **Languages** | Python 3.11+, TypeScript, Go (Golang), POSIX Bash, SQL |
| **Backend & Architecture** | Distributed Microservices, Event-Driven Architecture, CQRS / Asynchronous Decoupling, FastAPI, Starlette, Pydantic v2, Dependency Injection, Concurrency (`asyncio`, daemon threads) |
| **AI Orchestration & Agents** | LangGraph, LangChain, ReAct Framework, Function/Tool Calling, OpenAI Realtime API, Gemini Live, Deepgram Voice Agent, Tiktoken BPE, Context Compaction & Summarization |
| **Databases & Storage** | PostgreSQL 16, pgvector (Векторный поиск, Cosine distance), Redis 7 (Sorted Sets, Transactions, Lua scripting), SQLAlchemy 2.0 ORM, SQLite, Row-Level Locking (`FOR UPDATE SKIP LOCKED`), PG `LISTEN / NOTIFY` |
| **Telephony & Real-Time Audio** | Asterisk 20+, PJSIP, ARI (Asterisk REST Interface), AudioSocket Protocol, RTP UDP streaming, Voice Activity Detection (VAD), Barge-In / Echo cancellation, Whisper STT, Piper/Kokoro TTS |
| **Security & Cryptography** | AES-256-GCM (AEAD Encryption), HMAC-SHA256 Signed URLs / Tokens, RBAC (Role-Based Access Control), Sensitive Data Redaction |
| **Frontend Engineering** | React 18, Vite, Tailwind CSS, State Management (Zustand-like custom stores), WebSockets client, Design Systems, Playwright E2E Testing, Vitest |
| **Integrations & Protocols** | Bitrix24 REST & Disk API, Telegram MTProto (Telethon), Telegram Bot API (Aiogram 3), Umnico Webhooks, 1C:Enterprise REST API, WooCommerce API |
| **DevOps & Infrastructure** | Docker, Docker Compose (Multi-container orchestration), Nginx (Reverse Proxy, Load Balancing `least_conn`, SSL / ACME), Linux Systemd, Host Networking |
