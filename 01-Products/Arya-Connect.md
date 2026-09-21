# Arya Connect

## Домен
connect.arya.az (планируется)

## Статус
🟡 Спецификация — разработка не начата

## Краткое описание
B2B SaaS: омниканальный AI-CRM. Бизнес общается с клиентами через WhatsApp,
веб-виджет, голос, SMS и email из одного окна. AI отвечает автоматически из базы
знаний компании; при низкой уверенности продолжает отвечать, но помечает диалог
для менеджера. Менеджер может перехватить диалог в любой момент.

Первые клиенты: **Payonix**, **Italdizain**.

---

# 1. Обзор продукта

## 1.1 Задача

Малый и средний бизнес в Азербайджане ведёт переписку с клиентами одновременно
в WhatsApp, на сайте, по телефону и по почте. Истории разрознены, ответы
дублируются, ночью и в выходные никто не отвечает. Arya Connect собирает все
каналы в один интерфейс и закрывает типовые вопросы автоматически.

## 1.2 Ключевые свойства

| Свойство | Описание |
|---|---|
| Омниканальность | WhatsApp, веб-виджет, голос (SIP), SMS, email — один диалог, одна история |
| Единая база знаний | Одна KB на арендатора, работает во всех каналах одинаково |
| AI-первый ответ | Ответ генерируется из KB, не из общих знаний модели |
| Мягкая эскалация | При низкой уверенности AI **продолжает отвечать** и одновременно помечает диалог |
| Перехват | Менеджер в любой момент забирает диалог; AI замолкает |
| Мультиязычность | AZ, RU, EN — язык определяется по входящему сообщению |

## 1.3 Принципиальное решение: эскалация не прерывает диалог

Классическая схема «не уверен → молчи и жди человека» оставляет клиента без
ответа на неопределённый срок. В Arya Connect при низкой уверенности AI
**всё равно отвечает** (с осторожной формулировкой) и ставит флаг менеджеру.
Клиент всегда получает ответ; менеджер вмешивается, если ответ был неточным.

Следствие для реализации: флаг `needs_attention` — это не блокировка очереди,
а сигнал в панель. Диалог не переходит в состояние ожидания автоматически.

## 1.4 Границы продукта

**Входит:** KB, AI-ответы, диалоги, панель менеджера, статистика, вебхуки.

**Не входит:** доставка сообщений (это [[Arya-Hub]]), отправка почты (это
Arya Mail), маршрутизация конечных пользователей экосистемы ARYA (это
[[Arya-Concierge]]), собственная CRM (Phase 5).

---

# 2. Онбординг арендатора

## 2.1 Шаги

1. **Регистрация** — email + пароль (Supabase Auth), создаётся `tenants` + владелец в `tenant_users`
2. **База знаний** — загрузка файлов или ввод текста; асинхронная индексация
3. **Подключение каналов** — минимум один
4. **Менеджеры** — приглашение по email
5. **Готово** — виджет и номера активны

## 2.2 Состояния арендатора

```
pending → active → suspended → archived
```

- `pending` — зарегистрирован, нет ни одного канала или пустая KB
- `active` — есть канал и хотя бы один проиндексированный документ
- `suspended` — неоплата или нарушение; входящие принимаются, AI не отвечает
- `archived` — мягкое удаление, данные сохраняются 90 дней

## 2.3 API онбординга

```http
POST /api/tenants
Authorization: Bearer <supabase_jwt>
Content-Type: application/json

{ "name": "Payonix", "slug": "payonix", "default_language": "az",
  "timezone": "Asia/Baku" }

201 Created
{ "id": "uuid", "slug": "payonix", "status": "pending" }
```

```http
POST /api/tenants/:tenantId/users
{ "email": "manager@payonix.az", "role": "manager" }

201 Created
{ "id": "uuid", "status": "invited" }
```

Роли: `owner` (биллинг, удаление), `admin` (каналы, KB, пользователи),
`manager` (диалоги, перехват), `viewer` (только чтение и статистика).

---

# 3. База знаний

## 3.1 Модель

Одна KB на арендатора. Внутри — документы (`kb_documents`), каждый режется на
фрагменты (`kb_chunks`) с векторным представлением. Поиск — по всей KB
арендатора, независимо от канала.

## 3.2 Поддерживаемые типы

| Тип | Расширения | Парсер | Лимит |
|---|---|---|---|
| Текст | — (ввод в UI) | — | 100 000 символов |
| FAQ | — (пары вопрос/ответ в UI) | — | 500 пар |
| PDF | `.pdf` | `unpdf` (Workers-совместим) | 20 МБ |
| Word | `.doc`, `.docx` | `mammoth` | 20 МБ |
| Excel | `.xls`, `.xlsx` | `xlsx` (SheetJS) | 20 МБ |

> ⚠️ Парсинг PDF/DOCX/XLSX **не выполняется в Worker**: библиотеки тяжёлые,
> а CPU-время Worker ограничено. Файл загружается в Supabase Storage, задача
> ставится в `kb_ingest_jobs`, обработку ведёт отдельный воркер (Cloudflare
> Queues consumer или контейнер). См. §3.6.

## 3.3 Чанкинг и эмбеддинги

- Размер фрагмента: **800 токенов**, перекрытие **100 токенов**
- Границы — по абзацам, затем по предложениям; таблицы XLSX — построчно
- FAQ: одна пара «вопрос + ответ» = один фрагмент, без разрезания
- Модель эмбеддингов: `text-embedding-004` (Google, 768 измерений)
- Языки: эмбеддинги мультиязычные — отдельные индексы для AZ/RU/EN **не нужны**

## 3.4 Схема

```sql
create table kb_documents (
  id            uuid primary key default gen_random_uuid(),
  tenant_id     uuid not null references tenants(id) on delete cascade,
  title         text not null,
  source_type   text not null check (source_type in
                  ('text','faq','pdf','doc','docx','xls','xlsx')),
  storage_path  text,                       -- путь в Supabase Storage
  language      text check (language in ('az','ru','en')),
  status        text not null default 'pending'
                  check (status in ('pending','processing','ready','failed')),
  error         text,
  chunk_count   integer not null default 0,
  created_by    uuid references tenant_users(id),
  created_at    timestamptz not null default now(),
  updated_at    timestamptz not null default now()
);

create table kb_chunks (
  id            uuid primary key default gen_random_uuid(),
  tenant_id     uuid not null references tenants(id) on delete cascade,
  document_id   uuid not null references kb_documents(id) on delete cascade,
  chunk_index   integer not null,
  content       text not null,
  token_count   integer,
  embedding     vector(768) not null,
  metadata      jsonb not null default '{}'::jsonb,  -- page, sheet, row, question
  created_at    timestamptz not null default now()
);

-- tenant_id первым: фильтр по арендатору должен отсекать до векторного поиска
create index idx_kb_chunks_tenant on kb_chunks (tenant_id, document_id);

-- HNSW быстрее IVFFlat на чтении и не требует обучения на данных
create index idx_kb_chunks_embedding on kb_chunks
  using hnsw (embedding vector_cosine_ops);
```

## 3.5 Поиск

Гибридный: векторный + полнотекстовый, объединение по Reciprocal Rank Fusion.
Чистый вектор плохо ищет артикулы, названия моделей и телефоны — для Italdizain
(мебель, артикулы) это критично.

```sql
create or replace function kb_search(
  p_tenant_id  uuid,
  p_query_emb  vector(768),
  p_query_text text,
  p_limit      integer default 8
) returns table (
  chunk_id uuid, document_id uuid, content text,
  metadata jsonb, score double precision
)
language sql stable security invoker
set search_path = public, pg_temp
as $$
  with vec as (
    select id, document_id, content, metadata,
           row_number() over (order by embedding <=> p_query_emb) as rnk
      from kb_chunks
     where tenant_id = p_tenant_id
     order by embedding <=> p_query_emb
     limit 30
  ),
  fts as (
    select id, document_id, content, metadata,
           row_number() over (
             order by ts_rank_cd(
               to_tsvector('simple', content),
               plainto_tsquery('simple', p_query_text)) desc) as rnk
      from kb_chunks
     where tenant_id = p_tenant_id
       and to_tsvector('simple', content) @@ plainto_tsquery('simple', p_query_text)
     limit 30
  )
  select coalesce(v.id, f.id),
         coalesce(v.document_id, f.document_id),
         coalesce(v.content, f.content),
         coalesce(v.metadata, f.metadata),
         coalesce(1.0 / (60 + v.rnk), 0) + coalesce(1.0 / (60 + f.rnk), 0)
    from vec v full outer join fts f on v.id = f.id
   order by 5 desc
   limit p_limit;
$$;
```

> `'simple'` вместо языковых словарей — в PostgreSQL нет словаря для
> азербайджанского, а `'russian'` исказил бы азербайджанские токены.

## 3.6 Конвейер индексации

```
POST /api/kb/documents (файл)
  → Supabase Storage
  → INSERT kb_documents (status='pending')
  → INSERT kb_ingest_jobs
  → 202 Accepted { document_id }

[воркер вне Cloudflare Workers]
  → claim job (FOR UPDATE SKIP LOCKED)
  → скачать, распарсить, разбить на фрагменты
  → эмбеддинги пачками по 100
  → INSERT kb_chunks
  → UPDATE kb_documents status='ready', chunk_count=N
```

```sql
create table kb_ingest_jobs (
  id           uuid primary key default gen_random_uuid(),
  tenant_id    uuid not null references tenants(id) on delete cascade,
  document_id  uuid not null references kb_documents(id) on delete cascade,
  status       text not null default 'pending'
                 check (status in ('pending','running','done','failed')),
  attempts     integer not null default 0,
  last_error   text,
  created_at   timestamptz not null default now(),
  started_at   timestamptz,
  finished_at  timestamptz
);

create index idx_kb_ingest_pending on kb_ingest_jobs (created_at)
  where status = 'pending';
```

---

# 4. Каналы

## 4.1 Общая модель

Канал — это пара «арендатор + внешний адрес». Все каналы приводятся к одному
внутреннему формату сообщения, поэтому логика AI не знает, откуда пришёл текст.

```sql
create table channels (
  id            uuid primary key default gen_random_uuid(),
  tenant_id     uuid not null references tenants(id) on delete cascade,
  type          text not null check (type in
                  ('whatsapp','widget','voice','sms','email')),
  status        text not null default 'pending'
                 check (status in ('pending','active','error','disabled')),
  external_id   text,        -- WABA phone_number_id / SIP trunk id / домен
  display_name  text,
  config        jsonb not null default '{}'::jsonb,
  secret_ref    text,        -- ключ в Cloudflare Secrets Store, НЕ сам секрет
  last_error    text,
  created_at    timestamptz not null default now(),
  unique (tenant_id, type, external_id)
);
```

> Секреты каналов (токены WABA, пароли SIP) **не хранятся в БД**. В `config`
> лежит только ссылка `secret_ref`; значение — в Cloudflare Secrets Store.

## 4.2 WhatsApp

Через [[Arya-Hub]] (Meta Cloud API). Бизнес подключает **собственный номер
WABA** — Arya выступает BSP.

Подключение:
1. Арендатор проходит Meta Embedded Signup → Arya получает `waba_id` и `phone_number_id`
2. Connect регистрирует канал в Hub: `POST /api/channels` (админ-токен Hub)
3. Hub настраивает вебхук на себя; Connect получает входящие **от Hub**, не от Meta

`config` для WhatsApp:
```json
{ "waba_id": "...", "phone_number_id": "...", "hub_channel_id": "uuid",
  "hub_tenant_id": "uuid", "display_phone": "+994..." }
```

## 4.3 Веб-виджет

Одна строка на сайте клиента:

```html
<script src="https://connect.arya.az/widget.js"
        data-key="wgt_live_a1b2c3..." async></script>
```

- `data-key` — публичный ключ виджета, привязан к `tenant_id` и списку доменов
- Скрипт проверяет `document.referrer` / `location.origin` против `allowed_origins`
- Транспорт: `POST /api/widget/messages` + Server-Sent Events для ответов
- Сессия посетителя — в `localStorage` (`arya_visitor_id`, UUID v4)

`config` для виджета:
```json
{ "allowed_origins": ["https://payonix.az"], "theme_color": "#1f6feb",
  "greeting": { "az": "Salam!", "ru": "Здравствуйте!", "en": "Hello!" },
  "position": "bottom-right" }
```

## 4.4 Голос (SIP)

Бизнес подключает **свой SIP-транк** — Caspian Telecom, Bakcell, собственная АТС.
Arya не перепродаёт телефонию.

Цепочка: `SIP-транк → медиасервер → STT → Connect AI → TTS → обратно в SIP`.

> ⚠️ Медиапоток (RTP) **невозможен в Cloudflare Workers**: нет UDP, нет
> долгоживущих сокетов. Нужен отдельный медиасервер (FreeSWITCH / Asterisk /
> LiveKit SIP) на обычном хосте. Worker получает от него только текст:
>
> ```http
> POST /api/voice/turn
> X-Media-Secret: <secret>
> { "tenant_id": "...", "call_id": "...", "from": "+994...",
>   "transcript": "Salam, sifarişim harada?", "language": "az" }
>
> 200 OK
> { "reply_text": "...", "end_call": false }
> ```

`config` для голоса:
```json
{ "sip_host": "sip.caspel.az", "sip_username": "...", "secret_ref": "...",
  "inbound_did": "+994121234567", "codec": "PCMU", "stt": "deepgram",
  "tts": "google", "voice_id": "az-AZ-Standard-A" }
```

## 4.5 SMS

Через [[Arya-Hub]], провайдер LSIM (`apps.lsim.az`). Односторонний по смыслу:
SMS используется для уведомлений и коротких ответов, длинные ответы AI
обрезаются до 3 сегментов (≈ 400 символов) со ссылкой на веб-чат.

## 4.6 Email

Через Arya Mail.

- **Исходящие:** `POST /api/v1/outreach`
- **Входящие:** Arya Mail принимает письмо (вебхук Resend), кладёт в
  `inbound_emails` и пересылает на зарегистрированный вебхук арендатора

> ⚠️ Маршрутизация входящей почты в Arya Mail работает **только** для адресов
> вида `reply+<userId>+<leadId>@<домен>`. Без этого шаблона письмо сохраняется,
> но не маршрутизируется (в логе `Cannot route: no subscriber ID in reply-to
> address`). Connect обязан:
> 1. регистрировать вебхук в Arya Mail (`POST /api/webhooks`, событие `inbound`);
> 2. отправлять исходящие с `Reply-To: reply+<tenant_id>+<conversation_id>@mail.arya.az`.
>
> На момент написания спецификации ни один продукт экосистемы вебхук не
> регистрирует — Connect будет первым.

---

# 5. Логика AI

## 5.1 Конечный автомат диалога

```
        ┌──────────────────────────────────────────┐
        ▼                                          │
   [ai_active] ──низкая уверенность──► [ai_active + needs_attention]
        │                                          │
        │ менеджер нажал «Перехватить»             │
        ▼                                          ▼
   [manager_active] ◄─────────────────────────────┘
        │
        │ менеджер нажал «Завершить»
        ▼
   [ai_active]   (или [closed] при явном закрытии)
```

Состояния (`conversations.state`):

| Состояние | AI отвечает | Флаг в панели |
|---|---|---|
| `ai_active` | да | нет |
| `manager_active` | **нет** | «у менеджера» |
| `closed` | нет | архив |

`needs_attention` — **отдельный boolean**, не состояние. Может быть `true` при
`ai_active`: AI ответил, но не уверен.

## 5.2 Обработка входящего сообщения

```
1.  Вебхук от Hub / виджета / Mail / медиасервера
2.  Проверка подписи или секрета → иначе 401
3.  Дедупликация по external_message_id → если дубль, 200 и выход
4.  Ответ 200 немедленно; дальнейшее — в ctx.waitUntil()
5.  Найти/создать conversation по (tenant_id, channel_id, contact_key)
6.  Записать message (direction='inbound')
7.  Если state='manager_active' → стоп (AI молчит), уведомить панель
8.  Определить язык (по тексту, затем по contact.language, затем tenant default)
9.  kb_search(tenant_id, embedding(text), text, 8)
10. Собрать промпт: system + KB-фрагменты + последние 10 сообщений
11. Вызов Gemini → ответ + self-reported confidence
12. Посчитать итоговую уверенность (§5.3)
13. Если < порога → needs_attention = true, записать причину
14. Отправить ответ через Hub / SSE / Mail
15. Записать message (direction='outbound', ai_generated=true)
16. Realtime-событие в панель
```

> ⚠️ Шаги 5–16 обязаны выполняться под `ctx.waitUntil()`. В Cloudflare Workers
> работа, начатая после `return`, прерывается вместе с изолятом.

## 5.3 Оценка уверенности

Не полагаемся на самооценку модели — она завышена. Итоговая оценка:

```
confidence = 0.5 * max_similarity        // лучший косинус из kb_search
           + 0.3 * coverage              // доля фрагментов выше порога 0.75
           + 0.2 * model_self_report     // 0..1, запрошенная у модели
```

Порог эскалации по умолчанию `0.62`, настраивается на арендатора
(`tenants.escalation_threshold`).

Безусловная эскалация (независимо от оценки):
- KB пуста или `kb_search` вернул 0 фрагментов
- Клиент просит человека (совпадение со списком фраз AZ/RU/EN)
- Третье подряд сообщение клиента по одной теме (признак непонимания)
- Обнаружены реквизиты оплаты, жалоба, юридическая угроза

## 5.4 Системный промпт

```
Ты — ассистент компании {tenant_name}. Отвечай ТОЛЬКО на основе
приведённых ниже фрагментов базы знаний.

Правила:
- Если ответа нет во фрагментах — скажи, что уточнишь у коллеги.
  НЕ придумывай.
- Отвечай на языке клиента: {language}.
- Не называй цены, сроки и условия, которых нет во фрагментах.
- Никогда не проси оплату и не диктуй реквизиты.
- Коротко: 2–4 предложения.

Фрагменты базы знаний:
{chunks}

История диалога:
{history}
```

Фильтр вывода (по образцу `sanitizeGeminiResponse` в Real Estate): ответ,
содержащий номер телефона рядом с инструкцией о переводе денег, заменяется на
безопасную формулировку.

## 5.5 Перехват и возврат

```http
POST /api/conversations/:id/takeover
→ state='manager_active', assigned_to=<user_id>, needs_attention=false

POST /api/conversations/:id/release
→ state='ai_active', assigned_to=null
```

Захват — атомарный, иначе два менеджера перехватят один диалог:

```sql
update conversations
   set state = 'manager_active', assigned_to = $2, needs_attention = false
 where id = $1 and tenant_id = $3 and state <> 'manager_active'
returning *;
-- 0 строк → 409 Conflict, диалог уже у другого менеджера
```

---

# 6. Панель менеджера

## 6.1 Интерфейс

- **Список диалогов:** все каналы вместе, сортировка по последнему сообщению
- **Фильтры:** канал, состояние (`ai` / `needs_attention` / `takeover` / `closed`), назначен мне, период
- **Окно диалога:** полная история с иконкой канала у каждого сообщения
- **Ответ:** отправка от имени менеджера в исходный канал
- **Статистика:** время ответа, доля решённых, топ вопросов

## 6.2 Realtime

Supabase Realtime, канал на арендатора:

```ts
supabase.channel(`tenant:${tenantId}`)
  .on('postgres_changes',
      { event: '*', schema: 'public', table: 'messages',
        filter: `tenant_id=eq.${tenantId}` }, onMessage)
  .on('postgres_changes',
      { event: 'UPDATE', schema: 'public', table: 'conversations',
        filter: `tenant_id=eq.${tenantId}` }, onConversation)
  .subscribe();
```

RLS обязана быть включена на обеих таблицах — Realtime уважает политики, и без
них арендатор подпишется на чужой поток.

## 6.3 Метрики

| Метрика | Расчёт |
|---|---|
| Время первого ответа | `first_outbound.created_at − first_inbound.created_at` |
| Доля автоответов | диалоги без `manager_active` / все диалоги |
| Доля эскалаций | диалоги с `needs_attention=true` / все |
| Топ вопросов | кластеризация входящих по эмбеддингам, еженедельно |
| Покрытие KB | доля ответов с `max_similarity ≥ 0.75` |

Агрегаты считаются по расписанию (Cron Trigger, каждые 15 минут) и пишутся в
`stats_daily` — панель не обязана сканировать `messages`.

---

# 7. CRM-коннектор

## 7.1 Вебхуки

```sql
create table webhooks (
  id          uuid primary key default gen_random_uuid(),
  tenant_id   uuid not null references tenants(id) on delete cascade,
  url         text not null,
  events      text[] not null default '{lead.created}',
  secret      text not null,          -- для HMAC-подписи
  is_active   boolean not null default true,
  created_at  timestamptz not null default now()
);

create table webhook_deliveries (
  id           uuid primary key default gen_random_uuid(),
  webhook_id   uuid not null references webhooks(id) on delete cascade,
  event        text not null,
  payload      jsonb not null,
  status       text not null default 'pending',
  attempts     integer not null default 0,
  response_code integer,
  created_at   timestamptz not null default now(),
  delivered_at timestamptz
);
```

События: `lead.created`, `conversation.escalated`, `conversation.closed`.

Подпись: `X-Arya-Signature: sha256=<hmac(secret, body)>`.
Повторы: 3 попытки, задержки 1 с / 10 с / 60 с, только на 5xx и таймауты.

## 7.2 Payload

```json
{
  "event": "lead.created",
  "tenant_id": "uuid",
  "occurred_at": "2026-09-21T10:15:00Z",
  "data": {
    "lead_id": "uuid",
    "conversation_id": "uuid",
    "channel": "whatsapp",
    "contact": { "name": "Elvin", "phone": "+994501234567", "email": null },
    "first_message": "Salam, qiymət soruşmaq istəyirəm",
    "language": "az"
  }
}
```

## 7.3 Готовые коннекторы (Phase 4)

- **Bitrix24** — `crm.lead.add` через входящий вебхук портала
- **HubSpot** — Contacts API v3, привязка по email/телефону

---

# 8. Технологический стек

| Слой | Технология |
|---|---|
| Backend | Hono + TypeScript на Cloudflare Workers |
| Frontend | React 19 + Vite + Tailwind, Cloudflare Pages |
| БД | Supabase PostgreSQL + pgvector |
| Realtime | Supabase Realtime |
| Файлы | Supabase Storage |
| AI | Google Gemini (`gemini-2.5-flash`), эмбеддинги `text-embedding-004` |
| Каналы | [[Arya-Hub]] (WhatsApp, SMS), Arya Mail (email), SIP-медиасервер (голос) |
| Очереди | Cloudflare Queues (индексация KB, доставка вебхуков) |
| Расписание | Cloudflare Cron Triggers (агрегаты, ретраи) |

## 8.1 Ограничения Cloudflare Workers, влияющие на архитектуру

Эти ограничения уже проверены на миграциях Press и Real Estate — их нужно
учитывать с первого дня, а не обнаруживать в процессе.

| Ограничение | Следствие для Connect |
|---|---|
| Нет TCP-сокетов | Только `supabase-js` (HTTP/PostgREST) или Hyperdrive. Драйверы `pg`/`postgres` не работают |
| Нет процесса между запросами | Никаких `setInterval`, in-memory Map, счётчиков в модуле — всё в БД |
| Работа после `return` убивается | Любая отложенная логика — только через `ctx.waitUntil()` |
| `db.transaction()` не поддерживается pg-proxy | Атомарность — через функции PostgreSQL (`SECURITY INVOKER`), не через транзакции в коде |
| CPU-лимит на запрос | Парсинг PDF/DOCX/XLSX — вне Worker (см. §3.6) |
| Нет UDP | Медиапоток SIP — вне Worker (см. §4.4) |
| Cron Trigger: минимум 1 минута | Опросы чаще раза в минуту невозможны; проектировать событийно |

---

# 9. Архитектура

## 9.1 Разделение ответственности

```
┌─────────────────┐   входящие    ┌──────────────────┐
│    Arya Hub     │ ────────────► │   Arya Connect   │
│  (доставка)     │ ◄──────────── │  (KB + AI + UI)  │
└─────────────────┘  POST /send   └──────────────────┘
        │                                  │
   WhatsApp/SMS                      Supabase
   Meta / LSIM                    (pgvector, Realtime)

┌─────────────────┐
│  Arya Mail      │ ◄── исходящие /api/v1/outreach
│  (почта)        │ ──► входящие через вебхук
└─────────────────┘

┌─────────────────┐
│ Arya Concierge  │  маршрутизатор для КОНЕЧНЫХ пользователей
│                 │  экосистемы ARYA — НЕ участвует в Connect
└─────────────────┘
```

## 9.2 Правила

1. **Connect никогда не доставляет сообщения сам.** Нет прямых вызовов Meta
   Graph API, нет SMTP, нет SIP-регистрации из Worker. Только Hub и Mail.
2. **Hub не содержит бизнес-логики Connect.** Hub не знает про KB, уверенность
   и перехват. Он доставляет байты.
3. **Concierge и Connect не пересекаются.** Concierge маршрутизирует физлиц
   между продуктами ARYA; Connect обслуживает клиентов бизнеса-арендатора.
   Общая у них только инфраструктура доставки — Hub.
4. **Арендатор Connect ≠ арендатор Hub.** В Hub у Connect один служебный
   `tenant`, внутри которого заводятся каналы всех арендаторов Connect.
   Сопоставление хранится в `channels.config.hub_channel_id`.

## 9.3 Риск: Hub — единая точка отказа

Весь WhatsApp- и SMS-трафик экосистемы идёт через Hub, а его очереди отправки
(`greenApiReplyQueues`, `proactiveQueue`) живут **в памяти процесса** и теряются
при перезапуске. Для Connect это означает:

- исходящие сообщения нужно записывать в `messages` со `status='queued'`
  **до** вызова Hub и переводить в `sent` только по успешному ответу;
- Cron Trigger раз в минуту повторяет отправку зависших `queued` старше 2 минут;
- при недоступности Hub диалог не теряется, клиент получает ответ с задержкой.

---

# 10. Интеграции

## 10.1 Arya Hub — исходящие

```http
POST https://wa.arya.az/api/send
Authorization: Bearer <WA_HUB_TOKEN>
Content-Type: application/json

{
  "phone": "+994501234567",
  "channelId": "<hub_channel_id>",
  "message": "Sifarişiniz hazırdır.",
  "type": "reply"
}
```

`type`: `reply` — ответ внутри 24-часового окна; `outbound` — инициирующее
сообщение (для WhatsApp требует утверждённый шаблон Meta).

## 10.2 Arya Hub — входящие

Hub присылает на `forwardUrl` арендатора конверт с заголовком
`X-Router-Secret: <ROUTER_SECRET>`:

```http
POST https://connect.arya.az/api/webhooks/hub
X-Router-Secret: <ROUTER_SECRET>

{
  "phone": "+994501234567",
  "message": "Salam",
  "channelId": "uuid",
  "replyUrl": "https://wa.arya.az/api/send",
  "replyToken": "...",
  "idMessage": "wamid...."
}
```

> ⚠️ `replyToken` — **не идентификатор сообщения**. Hub присылает одну и ту же
> константу на каждый запрос; дедупликация по нему отбросит все сообщения,
> кроме первого. Дедуплицировать нужно по `idMessage`, а при его отсутствии —
> по `(phone, hash(message), окно 5 секунд)`.

Ответ обязан быть `200 OK` немедленно — Hub повторяет запрос на любой не-2xx.

## 10.3 Arya Mail — исходящие

```http
POST https://arya-mail.replit.app/api/v1/outreach
Authorization: Bearer <ARYA_MAIL_API_KEY>

{
  "to": "client@example.com",
  "subject": "Re: Ваш запрос",
  "body": "...",
  "leadId": "conn_<conversation_id>",
  "type": "outreach",
  "tags": { "source": "connect", "tenant_id": "uuid" }
}
```

## 10.4 Arya Mail — входящие

```http
POST https://arya-mail.replit.app/api/webhooks
Authorization: Bearer <ARYA_MAIL_API_KEY>

{ "url": "https://connect.arya.az/api/webhooks/mail",
  "events": ["inbound"] }
```

См. предупреждение в §4.6 про формат `reply+<...>@`.

## 10.5 SIP

Стандартный SIP-транк на арендатора. Connect не терминирует SIP; медиасервер
обращается к `POST /api/voice/turn` (§4.4).

---

# 11. Мультитенантность

## 11.1 Правило

`tenant_id` — на **каждой** таблице с пользовательскими данными, включая
`kb_chunks`, `messages`, `webhook_deliveries`. Без исключений: отсутствие
колонки на одной таблице ломает изоляцию всей модели.

## 11.2 Row Level Security

RLS включается на всех таблицах арендатора. Worker ходит под service-role
(RLS обходится) — **изоляция в API обеспечивается кодом**, явным
`.eq('tenant_id', tenantId)` в каждом запросе. RLS защищает второй контур:
Realtime-подписки и прямой доступ фронтенда под анонимным ключом.

```sql
alter table conversations enable row level security;

create policy tenant_isolation_conversations on conversations
  for all to authenticated
  using (tenant_id = (auth.jwt() ->> 'tenant_id')::uuid)
  with check (tenant_id = (auth.jwt() ->> 'tenant_id')::uuid);
```

`tenant_id` попадает в JWT через Supabase Auth custom claims при входе.

## 11.3 Ядро схемы

```sql
create table tenants (
  id                   uuid primary key default gen_random_uuid(),
  slug                 text not null unique,
  name                 text not null,
  status               text not null default 'pending'
                         check (status in ('pending','active','suspended','archived')),
  default_language     text not null default 'az'
                         check (default_language in ('az','ru','en')),
  timezone             text not null default 'Asia/Baku',
  escalation_threshold double precision not null default 0.62,
  plan                 text not null default 'trial',
  created_at           timestamptz not null default now()
);

create table tenant_users (
  id          uuid primary key default gen_random_uuid(),
  tenant_id   uuid not null references tenants(id) on delete cascade,
  auth_uid    uuid unique,                 -- Supabase Auth user id
  email       text not null,
  role        text not null check (role in ('owner','admin','manager','viewer')),
  status      text not null default 'invited'
                check (status in ('invited','active','disabled')),
  created_at  timestamptz not null default now(),
  unique (tenant_id, email)
);

create table contacts (
  id          uuid primary key default gen_random_uuid(),
  tenant_id   uuid not null references tenants(id) on delete cascade,
  name        text,
  phone       text,
  email       text,
  language    text check (language in ('az','ru','en')),
  metadata    jsonb not null default '{}'::jsonb,
  created_at  timestamptz not null default now()
);

create index idx_contacts_tenant_phone on contacts (tenant_id, phone);
create index idx_contacts_tenant_email on contacts (tenant_id, email);

create table conversations (
  id               uuid primary key default gen_random_uuid(),
  tenant_id        uuid not null references tenants(id) on delete cascade,
  channel_id       uuid not null references channels(id) on delete cascade,
  contact_id       uuid references contacts(id) on delete set null,
  -- телефон / visitor_id / email — по чему склеиваем диалог в канале
  contact_key      text not null,
  state            text not null default 'ai_active'
                     check (state in ('ai_active','manager_active','closed')),
  needs_attention  boolean not null default false,
  attention_reason text,
  assigned_to      uuid references tenant_users(id) on delete set null,
  language         text check (language in ('az','ru','en')),
  last_message_at  timestamptz not null default now(),
  created_at       timestamptz not null default now(),
  closed_at        timestamptz,
  unique (tenant_id, channel_id, contact_key)
);

create index idx_conv_tenant_recent on conversations (tenant_id, last_message_at desc);
create index idx_conv_attention on conversations (tenant_id, last_message_at desc)
  where needs_attention = true;

create table messages (
  id                  uuid primary key default gen_random_uuid(),
  tenant_id           uuid not null references tenants(id) on delete cascade,
  conversation_id     uuid not null references conversations(id) on delete cascade,
  direction           text not null check (direction in ('inbound','outbound')),
  body                text,
  media_url           text,
  ai_generated        boolean not null default false,
  author_id           uuid references tenant_users(id) on delete set null,
  confidence          double precision,
  kb_chunk_ids        uuid[],              -- на чём основан ответ (аудит)
  external_message_id text,                -- idMessage Hub / Resend id
  status              text not null default 'sent'
                        check (status in ('queued','sent','delivered','read','failed')),
  error               text,
  created_at          timestamptz not null default now()
);

create index idx_messages_conv on messages (conversation_id, created_at);
-- дедупликация входящих: один и тот же external id не обрабатывается дважды
create unique index idx_messages_external on messages (tenant_id, external_message_id)
  where external_message_id is not null;
```

## 11.4 Лимиты на арендатора

| Ресурс | Trial | Business |
|---|---|---|
| Документы KB | 10 | 500 |
| Фрагменты KB | 2 000 | 100 000 |
| Сообщений в месяц | 1 000 | 50 000 |
| Каналов | 2 | не ограничено |
| Менеджеров | 2 | 25 |

Счётчик сообщений — в `usage_counters` (арендатор + месяц), инкремент одним
`insert ... on conflict do update`, а не чтением-записью.

---

# 12. Дорожная карта

| Фаза | Объём | Результат |
|---|---|---|
| **1** | WhatsApp + виджет + KB + AI + панель | Payonix и Italdizain в проде |
| **2** | Голос (SIP) + SMS | Медиасервер, транк Caspian Telecom |
| **3** | Email | Регистрация вебхука в Arya Mail, схема `reply+` |
| **4** | CRM-коннекторы | Bitrix24, затем HubSpot |
| **5** | Собственный модуль CRM | Сделки, воронка, задачи |

## 12.1 Критерии готовности Phase 1

- [ ] Арендатор регистрируется и подключает WhatsApp без участия разработчика
- [ ] KB принимает PDF/DOCX/XLSX и индексируется за < 5 минут на 20 МБ
- [ ] AI отвечает на AZ/RU/EN, время ответа p95 < 4 с
- [ ] Эскалация срабатывает и видна в панели < 2 с (Realtime)
- [ ] Перехват атомарен: два менеджера не могут забрать один диалог
- [ ] Изоляция арендаторов проверена тестом: запрос чужого `tenant_id` → 403

---

# 13. Открытые вопросы

1. **Meta BSP.** Подключение чужих номеров WABA через Embedded Signup требует
   одобренного BSP-статуса. Заявка Arya Hub подана 07.08 — до одобрения
   Phase 1 возможна только на номерах, заведённых вручную.
2. **Медиасервер для голоса.** Не выбран (FreeSWITCH / Asterisk / LiveKit SIP)
   и не определён хостинг — вне Cloudflare.
3. **Воркер индексации KB.** Cloudflare Queues consumer имеет те же лимиты CPU,
   что и Worker; для 20 МБ PDF, вероятно, нужен контейнер.
4. **Arya Mail inbound.** Схема `reply+<userId>+<leadId>@` предполагает
   `userId` подписчика Mail. Нужно решить, как отобразить `tenant_id` Connect
   на эту схему, и подтвердить, что `webhook_configs` заполняется.
5. **Биллинг.** Не специфицирован: провайдер (Stripe / Kapital), тарифы, учёт
   перерасхода сообщений.
6. **Хранение диалогов.** Срок хранения и требования ПДн Азербайджана не
   определены.

---

## Связанные документы
[[Arya-Hub]] · [[Arya-Concierge]] · [[Arya-Voice]]

## Изменения
- 2026-09-21 — первая редакция спецификации
