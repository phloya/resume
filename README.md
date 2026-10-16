# Далер Хамидов — Senior AI Engineer

> **Резюме и инженерное портфолио**  
> 🌐 **Сайт:** [khamidovdaler.com](https://khamidovdaler.com)  
> 🔗 **GitHub Pages Mirror:** [phloya.github.io/resume](https://phloya.github.io/resume/)  
> 📄 **PDF-версия:** [Скачать PDF](daler-khamidov-resume.pdf)  
> 💬 **Telegram:** [@sutekinamirai](https://t.me/sutekinamirai)  
> ✉️ **Email:** [khamidovdaler24@gmail.com](mailto:khamidovdaler24@gmail.com)  

---

## 📌 Профиль

**Senior AI Engineer** с опытом проектирования и вывода в production распределенных AI-платформ, backend-систем, голосовой телефонии и real-time сервисов.

**Ключевой фокус:**
- Агентные архитектуры на **LangGraph** (персистентность графов, tool calling, structured output, human-in-the-loop).
- **Voice AI & Asterisk** телефония (ARI, AudioSocket, RTP, WebRTC, потоковый STT/TTS с задержкой **450–700 мс**, VAD, barge-in).
- Отказоустойчивый распределенный backend (**FastAPI**, **asyncio**, очереди задач, dead-letter queues, idempotent retry policies).
- Мультитенантные SaaS-системы с изоляцией данных (**PostgreSQL RLS**, transactional outbox).
- RAG и поиск по базам знаний (**pgvector**, **FAISS**, гибридный поиск, графы знаний).
- Интеграции: **Bitrix24**, **1С:Предприятие**, **Telegram Bot/MTProto**, **WhatsApp Cloud API**, **Instagram Graph API**.

---

## 📂 Технические досье и архитектурные аудиты проектов

В репозитории доступны детальные инженерные аудиты по каждому ключевому продукту:

1. 🚀 [Hub — Омниканальная AI-платформа](airoporthub.md)
2. 🎙️ [Голосовой AI-агент для телефонии (AVA)](telephony.md) — *>205 000 LOC, Asterisk, ARI, RTP, WebSockets*
3. 💬 [AI-ассистент для продаж](aisha.md) — *FastAPI, LangGraph, Telethon, Bitrix24*
4. 🛡️ [Control — SaaS-платформа поддержки и SLA](kontrol.md) / [saas.md](saas.md) — *PostgreSQL RLS, Transactional Outbox*
5. 📊 [Платформа анализа закупок (Pulsar)](pulsar.md) — *13 источников, Knowledge Graph, pgvector, FAISS*
6. 📈 [Speech QA — Речевая аналитика и контроль качества](transcribe.md) — *Диаризация, чек-листы, детерминированные скоры*
7. 🌐 [Realtime Translation Subtitles](subtitle.md) — *WebRTC, Web Audio API, SSE, OBS Studio (500–800 мс)*
8. 💻 [Web Platform — Web-продукты и AI-интерфейсы](site.md) — *React, TypeScript, WebGL, генерация PDF*
9. 📑 [Комплексный технический аудит](technical_audit.md)

---

## 🛠️ Настройка DNS для домена `khamidovdaler.com`

Чтобы домен `khamidovdaler.com` открывал этот сайт через GitHub Pages:

В панели управления доменом (**Spaceship** / **Cloudflare** / **DNS Provider**) добавьте 4 **A-записи** для apex-домена `@`:

| Тип | Имя / Хост | Значение (IP-адрес) |
|---|---|---|
| **A** | `@` (или пусто) | `185.199.108.153` |
| **A** | `@` (или пусто) | `185.199.109.153` |
| **A** | `@` (или пусто) | `185.199.110.153` |
| **A** | `@` (или пусто) | `185.199.111.153` |

*(Опционально)* Для поддомена `www`:
| Тип | Имя / Хост | Значение |
|---|---|---|
| **CNAME** | `www` | `phloya.github.io.` |

GitHub Pages автоматически выпустит бесплатный SSL/TLS сертификат (Let's Encrypt).
